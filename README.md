# Loyalty Orders Processing Flow

This repository contains an Apache NiFi flow (LoyaltyOrders.json) designed to automate order and payment confirmation processes for a loyalty program system. The flow interacts with various internal and external APIs to manage order statuses, confirm payments, update loyalty points, and communicate with customers.

## 📋 Overview
The NiFi flow is structured into three main process groups, each handling a distinct part of the loyalty order lifecycle:

LoyaltyOrders-PaymentService/code-verify: Handles the verification of payment confirmation codes.

set-expired: Manages the expiration of pending confirmations.

burn-bonuses: Processes successful payments, updates order statuses, burns loyalty points, and triggers customer notifications.

The flow integrates with internal services (e.g., http://10.160.20.73:8089) and external loyalty platforms.

## 🏗️ Architecture & Process Groups
### 1. LoyaltyOrders-PaymentService/code-verify
This group polls for new payment confirmations, checks if a code has been entered, validates it, and updates the status of the order and payment accordingly.

### Key Processors:

/async/confirm/list-new (InvokeHTTP): Fetches a list of new confirmations awaiting action.

SplitJson: Splits the list of confirmations into individual flow files for processing.

EvaluateJsonPath: Extracts attributes like confirmation_short_id, send_code, entered_code, etc.

RouteOnAttribute: Routes flowfiles based on whether a code has been entered (isCodeEntered).

UpdateAttribute: Dynamically calculates the confirmation status (confirmation_verified, confirmation_pending, confirmation_expired) and tracks attempt counts.

/async/confirm/update (InvokeHTTP): Updates the confirmation status on the remote server.

/api/payment/update (InvokeHTTP): Updates the overall payment status.

/api/order/update (InvokeHTTP): Updates the final order status.

### 2. set-expired
This group periodically checks for sent confirmations that have passed their expiration time and marks them as expired.

## Key Processors:

InvokeHTTP (GET): Calls /api/payment/async/confirm/list-new to find confirmations with confirmation_send status.

SplitJson & EvaluateJsonPath: Processes the list and extracts confirmation_short_id and expiration_time.

UpdateAttribute: Calculates the action attribute, setting it to 'confirmation_expired' if the current time is past the expiration time.

InvokeHTTP (POST): Calls /api/payment/async/confirm/update to report the expired status back to the server.

3. burn-bonuses
This is the most complex group, handling the post-payment process: burning loyalty points, updating multiple statuses, adding tags to customer profiles, and sending SMS notifications.

## Key Processors:

/api/payment/list-new (InvokeHTTP): Fetches payments with a confirmation_verified status.

SplitJson & EvaluateJsonPath: Processes payments, extracting details like order_short_id, payment_short_id, client_phone, and amount.

RouteOnAttribute: Routes flow based on API response status codes (e.g., handles error -6000).

/api/order/search (InvokeHTTP): Fetches detailed order information (e.g., store details, commentary).

/gifts/purchases/new (InvokeHTTP): Burns loyalty points for the purchased gift.

UpdateAttribute & ReplaceText: Prepares payloads for status updates and notifications.

/api/order/update & /api/order/status/add (InvokeHTTP): Updates the order system.

/api/payment/update & /api/payment/status/add (InvokeHTTP): Updates the payment system.

/users/tags/add/ (InvokeHTTP): Adds a tag to the customer's loyalty profile.

send/sms-code (InvokeHTTP): Sends an SMS confirmation to the customer.

## ⚙️ Configuration
Environment Variables / Parameters
The flow uses Expression Language for configuration. Key attributes to configure (likely via Parameter Contexts in NiFi):

host: Base URL for internal APIs (e.g., http://10.160.20.73).

port: Port for internal APIs (e.g., 8089).

sp_token: API token for authenticating with the SailPlay service.

store_department_id: The department ID for the SailPlay API calls.

Controller Services
The flow does not currently define any Controller Services (like SSLContextService). These would need to be created and linked if connecting to HTTPS endpoints requiring custom SSL configuration.

## 🚀 Deployment
Import Template: Import the LoyaltyOrders.json file into your NiFi instance as a template or by dragging the JSON onto the canvas.

Configure Parameters: Create a Parameter Context defining the variables mentioned above (host, port, sp_token, store_department_id) and link it to the root process group.

Configure Controller Services: If needed, create and enable any necessary Controller Services (e.g., for SSL).

Start Processors: Enable the GenerateFlowFile processors (or their parent process groups) to begin the scheduled execution.

## 🔧 Monitoring
Use the NiFi UI to monitor the status of connections (queue sizes), processors, and view bulletins.

Key connections have Back Pressure set to 10000 objects or 1 GB to prevent memory overload.

Processors are configured with appropriate penaltyDuration and yieldDuration.

## 📝 Notes
Scheduling: The GenerateFlowFile components are set to run every 5 sec to trigger the polling process. Adjust this based on your performance and business requirements.

Error Handling: The flow primarily uses relationship-based routing (e.g., Failure, Retry). The Retry mechanism is configured on processors (retryCount=10, backoffMechanism="PENALIZE_FLOWFILE"). Ensure failure relationships are connected to appropriate handling logic (e.g., notifying operators, retrying later).

Sensitive Data: Passwords in processors (e.g., in InvokeHTTP) are not shown in the template and must be set after import, preferably using sensitive dynamic properties or Parameter Contexts.

Time Zones: The flow uses Europe/Moscow timezone for date conversions. Ensure your NiFi instance's timezone is configured correctly or adjust these expressions.

## 🧩 Dependencies
Apache NiFi 2.3.0+ (or compatible version)

Internal Order/Payment API: Access to the REST API running on the configured host and port.

NARs: Uses standard NiFi NARs (nifi-standard-nar, nifi-update-attribute-nar).
