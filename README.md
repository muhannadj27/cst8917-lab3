# FleetBook — Vehicle Booking System

CST8917 — Serverless Applications | Lab 3

FleetBook is a serverless vehicle booking system built with Azure Service Bus, Azure Logic Apps, and Azure Functions. A customer submits a booking through a web app, the request flows through a Service Bus Queue, gets processed by a Logic App that checks fleet availability and pricing via an Azure Function, and the customer receives a confirmation or rejection email. Results are published to a Service Bus Topic with filtered subscriptions for downstream routing.

## Architecture

**Web App → Service Bus Queue → Logic App → Azure Function → Condition (Confirmed/Rejected) → Email + Service Bus Topic**

- **Service Bus Queue (`booking-requests`)** — receives incoming booking requests from the web client
- **Azure Function (`check-booking`)** — evaluates fleet availability and calculates pricing (rental days, add-ons, weekly discount)
- **Logic App (`process-booking`)** — orchestrates the workflow: decodes and parses the booking, calls the function, branches on the result, sends the appropriate email, and publishes the outcome
- **Service Bus Topic (`booking-results`)** — publishes confirmed/rejected outcomes
- **Filtered Subscriptions (`confirmed-sub`, `rejected-sub`)** — route messages by label using SQL filters (`sys.label = 'confirmed'` / `'rejected'`)
- **Outlook connector** — sends confirmation/rejection emails to the customer
- **Web Client (`client.html`)** — single-file HTML app for submitting bookings and tracking results in real time

## How It Works

1. Customer submits a booking through `client.html`, which sends a message to the Service Bus queue
2. The Logic App trigger picks up the message, decodes it from base64, and parses the booking JSON
3. The Logic App calls the `check-booking` Azure Function, which checks fleet telematics data (availability, location, mileage) and calculates pricing
4. A Condition action branches on the function's response:
   - **Confirmed** → sends a confirmation email with pricing details, publishes to the topic with label `confirmed`
   - **Rejected** → sends a rejection email with the reason, publishes to the topic with label `rejected`
5. The web client polls the `confirmed-sub` and `rejected-sub` subscriptions and updates the booking dashboard in real time

## Project Structure

| File | Description |
|---|---|
| `function_app.py` | Azure Function — fleet availability check and pricing logic |
| `requirements.txt` | Python dependencies |
| `test-function.http` | REST Client test requests for local testing |
| `client.html` | FleetBook web app (no build step — open directly in browser) |
| `local.settings.example.json` | Settings template — copy to `local.settings.json` and fill in your own values before running locally |
| `host.json` | Azure Functions host configuration |

## Setup Instructions

### Prerequisites
- Azure subscription
- Azure Service Bus namespace (Standard tier — required for topics)
- Python 3.11 or 3.12
- Azure Functions Core Tools
- VS Code with the Azure Functions extension

### 1. Azure Service Bus
- Create a Service Bus namespace (Standard tier)
- Create queue `booking-requests`
- Create topic `booking-results`
- Create subscriptions `confirmed-sub` and `rejected-sub` with SQL filters on `sys.label`

### 2. Azure Function
```bash
python -m venv .venv
.venv\Scripts\activate
pip install -r requirements.txt --break-system-packages
```
Copy `local.settings.example.json` to `local.settings.json` and fill in your local values. Run locally with `func start`, then deploy to Azure via VS Code (`Azure Functions: Deploy to Function App`).

### 3. Logic App
Build the workflow in the Azure Portal Logic App Designer with:
- Service Bus queue trigger on `booking-requests`
- Decode (base64) + Parse JSON on the incoming booking
- Call the deployed Azure Function
- Parse JSON on the function response
- Condition on `status == "confirmed"`
- Each branch sends an email (Outlook/Office 365 connector) and publishes to the `booking-results` topic with the appropriate label

### 4. Web Client
Open `client.html` directly in a browser. In the Service Bus Configuration panel, enter your namespace name and SAS primary key (Azure Portal → Service Bus namespace → Shared access policies → RootManageSharedAccessKey).

## Demo Video

https://youtu.be/5ZZkPTQQPO0

The video demonstrates:
- Service Bus queue, topic, and filtered subscriptions in the Azure Portal
- A confirmed booking submitted through the web app, with the Logic App run showing the True branch and the confirmation email received
- A rejected booking submitted through the web app, with the Logic App run showing the False branch and the rejection email received
- Topic subscription message counts reflecting both outcomes

## Security Note

In production, separate least-privilege Service Bus access policies would be used instead of the root management key, and the SAS key would never be exposed in client-side code — a backend API would generate tokens on behalf of the client. This lab uses the root key and direct REST calls for simplicity.
