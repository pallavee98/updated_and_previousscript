## Previous Script
```
pallavee@pallavee:~$ cat updated_alert_webhook.py 
from flask import Flask, request, jsonify
import requests
from datetime import datetime, timedelta
import itertools

app = Flask(__name__)

# Redmine API Configuration
REDMINE_API_URL = 'http://localhost:3100/issues.json'  # Redmine server's IP and port inside the container
REDMINE_API_KEY = '2e914e337c97545ce6c79567ca24035087952506'

# File to store mapping of alert names to Redmine issue IDs
ISSUE_MAP_FILE = 'issues_map.txt'

# List of members' IDs (Redmine User IDs) in the project
members = itertools.cycle([7, 8, 9])  # Replace with actual Redmine User IDs

def save_issue_id(alertname, issue_id):
    """Save the issue ID associated with an alert name."""
    print(f"Starting to save issue ID {issue_id} for alert {alertname}")
    with open(ISSUE_MAP_FILE, 'a') as f:
        f.write(f"{alertname}:{issue_id}\n")
    print(f"Finished saving issue ID {issue_id} for alert {alertname}")

def find_latest_issue_id(alertname):
    """Find the most recent issue ID associated with an alert name."""
    print(f"Starting to find the latest issue ID for alert {alertname}")
    try:
        with open(ISSUE_MAP_FILE, 'r') as f:
            lines = f.readlines()
            # Reverse the list to find the most recent issue ID
            for line in reversed(lines):
                name, id = line.strip().split(':')
                if name == alertname:
                    print(f"Found issue ID {id} for alert {alertname}")
                    return id
    except FileNotFoundError:
        print(f"No previous issues found for alert {alertname}")
        return None
    print(f"Could not find a matching issue ID for alert {alertname}")
    return None

def get_next_assignee():
    """Get the next member in the cycle to assign the issue."""
    return next(members)

def create_redmine_issue(alert):
    print(f"Starting to create a Redmine issue for alert {alert['labels'].get('alertname')}")
    headers = {
        'X-Redmine-API-Key': REDMINE_API_KEY,
        'Content-Type': 'application/json'
    }
    
    # Calculate start date and due date
    start_date = datetime.now().date().isoformat()  # Current date in YYYY-MM-DD format
    due_date = (datetime.now().date() + timedelta(days=1)).isoformat()  # Due date is start date + 1 day
    
    assignee_id = get_next_assignee()  # Get the next member to assign the issue
    
    issue_data = {
        "issue": {
            "project_id": 4,  # Project ID for 'monitoring'
            "subject": alert.get('annotations', {}).get('summary', 'No summary provided'),
            "description": alert.get('annotations', {}).get('description', 'No description provided'),
            "priority_id": 11,  # Priority ID based on your Redmine setup
            "status_id": 1,  # Set the issue status to "New" when created
            "assigned_to_id": assignee_id,  # Assign the issue to the next member
            "start_date": start_date,  # Start date
            "due_date": due_date  # Due date
        }
    }
    
    response = requests.post(REDMINE_API_URL, json=issue_data, headers=headers)
    
    # Log the Redmine API response
    print(f"Redmine response status: {response.status_code}")
    print(f"Redmine response content: {response.text}")
    
    if response.status_code == 201:
        issue_id = response.json().get('issue', {}).get('id')
        print(f"Successfully created Redmine issue with ID {issue_id} for alert {alert['labels'].get('alertname')}")
        return issue_id
    else:
        print(f"Failed to create Redmine issue for alert {alert['labels'].get('alertname')}")
        return None

def close_redmine_issue(issue_id):
    print(f"Starting to close Redmine issue with ID {issue_id}")
    url = f'http://localhost:3100/issues/{issue_id}.json'

    headers = {
        'X-Redmine-API-Key': REDMINE_API_KEY,
        'Content-Type': 'application/json'
    }
    issue_data = {
        "issue": {
            "status_id": 13  # Using ID 13 for 'Closed' status
        }
    }
    response = requests.put(url, json=issue_data, headers=headers)
    
    # Log the Redmine API response
    print(f"Redmine response status: {response.status_code}")
    print(f"Redmine response content: {response.text}")
    
    if response.status_code in [200, 204]:
        print(f"Successfully closed Redmine issue with ID {issue_id}")
        return True
    else:
        print(f"Failed to close Redmine issue with ID {issue_id}")
        return False

@app.route('/api/v1/alerts', methods=['GET', 'POST'])
def receive_alert():
    if request.method == 'POST':
        print("Starting to process received alert")
        alert_data = request.json
        print("Received alert:", alert_data)
        
        for alert in alert_data.get('alerts', []):
            alertname = alert['labels'].get('alertname')
            if alert['status'] == 'firing':
                print(f"Alert {alertname} is firing, creating a Redmine issue")
                # Create a new issue in Redmine
                issue_id = create_redmine_issue(alert)
                if issue_id:
                    save_issue_id(alertname, issue_id)
                    print(f"Created Redmine issue with ID {issue_id}")
                else:
                    print("Failed to create Redmine issue")
            elif alert['status'] == 'resolved':
                print(f"Alert {alertname} is resolved, closing the corresponding Redmine issue")
                # Find the most recent corresponding issue ID
                issue_id = find_latest_issue_id(alertname)
                if issue_id:
                    closed = close_redmine_issue(issue_id)
                    if closed:
                        print(f"Closed Redmine issue with ID {issue_id}")
                    else:
                        print(f"Failed to close Redmine issue with ID {issue_id}")
                else:
                    print("Could not find corresponding issue ID for alert")
        
        print("Finished processing received alert")
        return jsonify({'status': 'received'}), 200
    else:
        return "Webhook endpoint is running.", 200

if __name__ == '__main__':
    print("Starting webhook server")
    app.run(host='0.0.0.0', port=5100, debug=True)
    print("Webhook server is running")
```
