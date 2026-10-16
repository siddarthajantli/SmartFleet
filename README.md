# SmartFleet
## Project Overview
SmartFleet is an intelligent vehicle diagnostics platform designed to help fleet owners, drivers, and workshop engineers identify vehicle Diagnostic Trouble Codes (DTCs), understand potential root causes, and take appropriate corrective action.

The solution combines AWS IoT Core, Amazon DynamoDB, Amazon SQS, AWS Lambda, Amazon Bedrock, and Amazon SNS to build an event-driven diagnostic workflow.

The platform is designed for commercial vehicles such as trucks, intercity buses, transport buses, and mini-passenger vans.

The primary objectives are to:
- Collect vehicle telemetry and ECU diagnostic events.
- Maintain separate current and historical DTC records.
- Prevent duplicate analysis of an already-active DTC.
- Trigger diagnostic processing asynchronously through an event-driven architecture.
- Use Amazon Bedrock to analyze DTCs using vehicle context and diagnostic knowledge.
- Deliver actionable diagnostic guidance to fleet operators and workshop engineers.
- Provide a scalable foundation for fleet-wide monitoring and predictive maintenance.

## High-Level Architecture
###  flowchart TD
    
    A[Vehicle ECU / Simulator] --> B[AWS IoT Core]
    B --> C[IoT Rule]
    C --> D[DTC Processor Lambda]

    D --> E[(DynamoDB Current DTCs)]
    D --> F[(DynamoDB DTC History)]

    E --> G[DynamoDB Streams]
    G --> H[EventBridge Pipe]
    H --> I[Amazon SQS DTC Analysis Queue]

    I --> J[DTC Analyzer Lambda]
    J --> K[Amazon Bedrock]
    K --> L[Diagnostic Analysis]

    L --> M[Amazon SNS]
    M --> N[Driver Notifications]
    M --> O[Fleet / Workshop Notifications]

**Architecture note:** The diagram represents the intended event-driven flow. DynamoDB Streams and EventBridge Pipes are used to propagate qualifying DTC state changes to the SQS analysis queue. The processor and stream-filtering logic determine which changes qualify for analysis.

## Vehicle Data Collection
The first stage collects diagnostic events and telemetry from the vehicle ECU / a vehicle simulator.

AWS IoT Core provides the secure MQTT-based ingestion layer. Each vehicle is provisioned with its own device certificate and publishes messages to its assigned MQTT topic.

### Example topic (IoT Core Topic):
smartfleet/vehicles/TRUCK-001/dtc
### Example DTC event (Subject to change):

~~~ json
{
  "vehicleId": "TRUCK-001",
  "vehicleType": "TRUCK",
  "vinNumber": "TEST_VIN",
  "eventId": "evt-20261016-0001",
  "timestamp": "2026-10-16T09:00:00Z",
  "dtc": {
    "code": "P0087",
    "status": "ACTIVE"
  },
  "telemetry": {
    "rpm": 2400,
    "speed": 65
  }
}
~~~

## The collected information includes:
- Vehicle identity and type.
- Vehicle identification number (VIN).
- Unique event identifier.
- Event timestamp.
- DTC code and status.
- Available telemetry, such as engine RPM and vehicle speed.

The diagnostic severity and likely root causes are determined during the analysis stage rather than being assumed to originate from the ECU event.

## Event Ingestion Through AWS IoT Core
AWS IoT Core receives the MQTT message and evaluates it against an IoT Rule.
The rule forwards qualifying messages to the DTC Processor Lambda function.

### Responsibilities of the ingestion layer include:
1. Receive the vehicle event.
2. Validate required fields.
3. Normalize the DTC code and status.
4. Preserve the event identifier and timestamp.
5. Forward the event for state management.

This decouples vehicle communication from diagnostic processing. Vehicles do not need to wait for an AI analysis to finish before sending their next event.

## DTC Processing and State Management
The DTC Processor Lambda function determines how the incoming event affects the vehicle's DTC state.
The system maintains two logical categories of DTC data.
1. Current DTCs
2. Historical DTCs

### Current DTCs
The current DTC store represents the latest known state of each DTC for a vehicle.
Example logical key:

PK (Partition Key) = VEHICLE#TRUCK-001
SK (Sort Key)= DTC#P0087

The current record can contain the DTC status, first-seen timestamp, last-seen timestamp, most recent event identifier, and other relevant fields.
Its purpose is to answer:
- Which DTCs are currently active?
- When was a DTC first detected?
- When was it last observed?
- Has the DTC been cleared?
- Does this event represent a new diagnostic-analysis opportunity?

### Historical DTCs
The historical store retains individual DTC lifecycle events and state transitions.
Example logical key:
PK = VEHICLE#TRUCK-001#DTC#P0087
SK = EVENT#2026-10-16T09:00:00Z#evt-20261016-0001

Historical events can record:
- DTC activation.
- Repeated observations.
- DTC clearance.
- Reappearance after clearance.
- Relevant telemetry snapshots.
- Event timestamps and identifiers.

The history supports troubleshooting, auditing, trend analysis, and future maintenance insights.
The logical keys above are illustrative; the deployed table key schema may differ.

## Duplicate Prevention and DTC Lifecycle
A DTC can be reported repeatedly while it remains active. Every repeated message should not automatically trigger another expensive AI analysis.
The DTC Processor applies lifecycle and deduplication rules.

| Incoming event      | Current state           | Expected behavior                                                                       |
|---------------------|-------------------------|-----------------------------------------------------------------------------------------|
| New ACTIVE DTC      | No active record        | Create/update current state and record history                                          |
| Repeated ACTIVE DTC | Already active          | Update observation details and retain history as required; avoid unnecessary reanalysis |
| CLEAR event         | Active record exists    | Mark the DTC cleared and record the transition                                          |
| ACTIVE after CLEAR  | Previously cleared      | Start a new active lifecycle and allow analysis again                                   |
| Duplicate event ID  | Event already processed | Avoid processing the same event twice                                                   |