# chalo


here its a basic solution without ansible , i have less walkthrough from ansible but had exposure to terraform a lot ..



Solution:- 

from flask import Flask, request, jsonify
import os
import subprocess
import json

app = Flask(__name__)

# Directory to store generated Terraform files
TERRAFORM_DIR = "./terraform"

# Initialize directory
# Create the directory if it doesn't exist to store Terraform configuration files
if not os.path.exists(TERRAFORM_DIR):
    os.makedirs(TERRAFORM_DIR)
    print(f"Created directory: {TERRAFORM_DIR}")

# Helper function to write Terraform files
def write_terraform_file(config):
    # Generate Terraform configuration for AWS provider, security group, primary and replica instances
    terraform_content = f'''
    provider "aws" {{
      region = "{config['region']}"
    }}

    resource "aws_security_group" "postgres_sg" {{
      name        = "postgres_security_group"

      # Allow inbound traffic on PostgreSQL default port (5432)
      ingress {{
        from_port   = 5432
        to_port     = 5432
        protocol    = "tcp"
        cidr_blocks = ["0.0.0.0/0"] # Update as per your security requirements
      }}

      # Allow all outbound traffic
      egress {{
        from_port   = 0
        to_port     = 0
        protocol    = "-1"
        cidr_blocks = ["0.0.0.0/0"]
      }}
    }}

    resource "aws_instance" "postgres_primary" {{
      ami                    = "{config['ami']}"
      instance_type          = "{config['instance_type']}"
      vpc_security_group_ids = [aws_security_group.postgres_sg.id]
      user_data              = "${file("./setup_primary.sh")}"  # User data script to set up primary PostgreSQL server
      tags = {{
        Name = "PostgresPrimary"
      }}
    }}

    resource "aws_instance" "postgres_replica" {{
      count                  = {config['number_of_replicas']}
      ami                    = "{config['ami']}"
      instance_type          = "{config['instance_type']}"
      vpc_security_group_ids = [aws_security_group.postgres_sg.id]
      user_data              = "${file("./setup_replica.sh")}"  # User data script to set up replica PostgreSQL servers
      tags = {{
        Name = "PostgresReplica-${{count.index}}"
      }}
    }}
    '''
    # Write the generated Terraform configuration to main.tf file
    with open(f"{TERRAFORM_DIR}/main.tf", "w") as tf_file:
        tf_file.write(terraform_content)
        print("Terraform configuration written to main.tf")

    # Write user data scripts for primary and replica setup
    # This script sets up the primary PostgreSQL server with necessary configurations
    with open(f"{TERRAFORM_DIR}/setup_primary.sh", "w") as primary_file:
        primary_file.write(f'''
        #!/bin/bash
        apt-get update
        apt-get install -y postgresql-{config['postgresql_version']}
        # Configure PostgreSQL to listen on all addresses
        echo "listen_addresses = '*'" >> /etc/postgresql/{config['postgresql_version']}/main/postgresql.conf
        # Set max_connections as per user input
        echo "max_connections = {config['max_connections']}" >> /etc/postgresql/{config['postgresql_version']}/main/postgresql.conf
        # Set shared_buffers as per user input
        echo "shared_buffers = '{config['shared_buffers']}MB'" >> /etc/postgresql/{config['postgresql_version']}/main/postgresql.conf
        # Allow replication from any host
        echo "host replication all 0.0.0.0/0 md5" >> /etc/postgresql/{config['postgresql_version']}/main/pg_hba.conf
        # Restart PostgreSQL service to apply changes
        systemctl restart postgresql
        ''')
        print("Primary setup script written to setup_primary.sh")

    # This script sets up the replica PostgreSQL server with necessary configurations for replication
    with open(f"{TERRAFORM_DIR}/setup_replica.sh", "w") as replica_file:
        replica_file.write(f'''
        #!/bin/bash
        apt-get update
        apt-get install -y postgresql-{config['postgresql_version']}
        PGDATA=/var/lib/postgresql/{config['postgresql_version']}/main
        # Remove existing data to prepare for replication setup
        rm -rf $PGDATA/*
        # Use pg_basebackup to create a replica of the primary database
        PGPASSWORD="{config['replication_password']}" pg_basebackup -h {config['primary_private_ip']} -D $PGDATA -U replication_user -v -P
        # Set up the replica to operate in standby mode
        echo "standby_mode = 'on'" >> $PGDATA/recovery.conf
        # Configure connection details for the primary server
        echo "primary_conninfo = 'host={config['primary_private_ip']} user=replication_user password={config['replication_password']}'" >> $PGDATA/recovery.conf
        # Restart PostgreSQL service to apply changes
        systemctl restart postgresql
        ''')
        print("Replica setup script written to setup_replica.sh")

# Endpoint to generate Terraform code
@app.route('/generate', methods=['POST'])
def generate_code():
    config = request.json
    print(f"Received configuration: {config}")
    try:
        # Generate Terraform and user data files based on the configuration provided by the user
        write_terraform_file(config)
        print("Terraform code generation successful")
        return jsonify({"message": "Terraform code generated successfully."}), 200
    except Exception as e:
        print(f"Error generating Terraform code: {e}")
        return jsonify({"error": str(e)}), 500

# Endpoint to run `terraform plan`
@app.route('/terraform-plan', methods=['POST'])
def terraform_plan():
    try:
        # Change directory to where the Terraform configuration is located
        os.chdir(TERRAFORM_DIR)
        print(f"Changed directory to: {TERRAFORM_DIR}")
        # Initialize Terraform (download necessary provider plugins, etc.)
        subprocess.run(["terraform", "init"], check=True)
        print("Terraform initialized successfully")
        # Run Terraform plan to see what changes will be made
        plan = subprocess.run(["terraform", "plan"], capture_output=True, text=True, check=True)
        print("Terraform plan executed successfully")
        os.chdir("../")  # Change back to the original directory
        print("Changed back to the original directory")
        return jsonify({"plan": plan.stdout}), 200
    except subprocess.CalledProcessError as e:
        os.chdir("../")  # Ensure we change back to the original directory even if an error occurs
        print(f"Error running terraform plan: {e.stderr}")
        return jsonify({"error": e.stderr}), 500

# Endpoint to run `terraform apply`
@app.route('/terraform-apply', methods=['POST'])
def terraform_apply():
    try:
        # Change directory to where the Terraform configuration is located
        os.chdir(TERRAFORM_DIR)
        print(f"Changed directory to: {TERRAFORM_DIR}")
        # Initialize Terraform (download necessary provider plugins, etc.)
        subprocess.run(["terraform", "init"], check=True)
        print("Terraform initialized successfully")
        # Apply the Terraform plan to create/update infrastructure
        apply = subprocess.run(["terraform", "apply", "-auto-approve"], capture_output=True, text=True, check=True)
        print("Terraform apply executed successfully")
        os.chdir("../")  # Change back to the original directory
        print("Changed back to the original directory")
        return jsonify({"apply": apply.stdout}), 200
    except subprocess.CalledProcessError as e:
        os.chdir("../")  # Ensure we change back to the original directory even if an error occurs
        print(f"Error running terraform apply: {e.stderr}")
        return jsonify({"error": e.stderr}), 500

if __name__ == '__main__':
    # Run the Flask application
    print("Starting Flask server on port 5000")
    app.run(debug=True, host='0.0.0.0', port=5000)
