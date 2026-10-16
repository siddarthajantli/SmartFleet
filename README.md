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

These rules distinguish an active fault from a new diagnostic event.
The implementation should use conditional writes, idempotency controls, and appropriate state-transition logic to prevent concurrent or retried events from creating duplicate analysis requests.

## Event-Driven Diagnostic Trigger
Once DynamoDB reflects a qualifying new active DTC, the system uses DynamoDB Streams to detect the change.
An EventBridge Pipe filters stream records and forwards only qualifying events to the Amazon SQS DTC Analysis queue.
The intended sequence is:
1. The DTC Processor writes the current DTC state to DynamoDB.
2. DynamoDB Streams captures the database change.
3. EventBridge Pipes filters the stream records.
4. A qualifying new active DTC is sent to SQS.
5. The DTC Analyzer Lambda consumes the queued message.

**Important:** DynamoDB does not directly push a message to SQS merely because an item exists. The stream and pipe provide the event-driven connection. Filtering should distinguish a new active lifecycle from ordinary updates and repeated observations.
This approach separates operational state management from AI analysis and allows the analysis workload to scale independently.

## Amazon SQS — Diagnostic Work Queue
Amazon SQS buffers DTC analysis requests between the event-processing layer and the diagnostic worker.
The queue provides:
- Asynchronous processing.
- Buffering during traffic spikes.
- Independent scaling of diagnostic workers.
- Retry handling for transient failures.
- Isolation between ingestion and AI processing.

The architecture uses a standard SQS queue rather than a FIFO queue.
The DTC Analyzer Lambda should be idempotent because messages may be delivered more than once. A dead-letter queue can be configured to retain messages that repeatedly fail processing.

## DTC Analyzer Lambda
The DTC Analyzer Lambda is responsible for converting a DTC event into a structured diagnostic request.
Its workflow includes:
1. Read the SQS message.
2. Validate the vehicle and DTC details.
3. Retrieve the relevant diagnostic context from the configured knowledge source.
4. Build the analysis prompt using the DTC, telemetry, and retrieved evidence.
5. Invoke the PowertrainAgent or configured Amazon Bedrock model.
6. Validate and record the response.
7. Send appropriate notifications when processing succeeds.

The analyzer must distinguish observed telemetry from assumptions. For example, an engine RPM of 2,400 does not by itself establish that a fuel pump has failed.

## Amazon Bedrock — Intelligent DTC Analysis

Amazon Bedrock provides the foundation-model capability for interpreting diagnostic information.
The SmartFleet PowertrainAgent uses Amazon Nova Lite and retrieves relevant documents from the SmartFleet Powertrain Knowledge Base through an Amazon Bedrock AgentCore Gateway MCP tool.

The diagnostic context can include:
- DTC code and status.
- Vehicle identity and type.
- Available telemetry.
- Retrieved diagnostic documentation.
- Relevant diagnostic procedures and checks.

The model generates a structured analysis containing plausible causes, supporting evidence, confidence estimates, and recommended checks.

The diagnostic approach should:
- Rank potential causes rather than assert an unconfirmed failure.
- Distinguish direct evidence from general diagnostic knowledge.
- Request additional information when necessary.
- Respect the specialist agent's diagnostic scope.
- Avoid fabricating sensor readings or test results.

For example, DTC P0087 indicates a fuel-rail/system pressure condition below the expected threshold. The analysis should consider multiple potential causes and recommend appropriate checks rather than automatically declaring the fuel pump defective.

## Knowledge Base and MCP Integration
The PowertrainAgent uses the SmartFleet diagnostic Knowledge Base to ground its reasoning in relevant documentation.
The local integration contains:

app/
└── PowertrainAgent/
    ├── main.py
    └── mcp_kb_client.py

**main.py** handles the diagnostic prompt, retrieves context through the MCP client, and passes the combined context to the Strands agent.
**mcp_kb_client.py** calls the AgentCore Gateway retrieval tool and extracts relevant Knowledge Base documents.

The diagnostic documentation includes general powertrain diagnostic procedures, fuel-pressure troubleshooting, catalyst and oxygen-sensor diagnostics, and DTC-specific guidance.
This retrieval-based design provides a foundation for explainable, document-grounded diagnostic recommendations.

## Amazon SNS — Driver and Workshop Notifications
After diagnostic analysis, Amazon SNS can distribute relevant results to configured subscribers. 
Potential notification destinations include:
- Fleet managers.
- Workshop engineers.
- Driver notification workflows.
- Operational email or SMS endpoints.

Notifications should summarize the DTC, potential causes, supporting evidence, recommended checks, and the urgency of the next action.
Safety-critical recommendations should be conservative and should not be based solely on an unverified model conclusion.
The severity and notification policy should be validated before production use.

## Monitoring and Operational Reliability
The architecture can be monitored using Amazon CloudWatch.
Useful operational metrics and logs include:
- IoT Rule delivery failures.
- DTC Processor Lambda errors and duration.
- DynamoDB throttling and conditional-write failures.
- DynamoDB Streams processing failures.
- EventBridge Pipe execution and delivery failures.
- SQS queue depth and message age.
- DTC Analyzer Lambda errors and duration.
- Bedrock invocation latency, throttling, and failures.
- SNS delivery failures.

Alarm thresholds should reflect expected vehicle event volumes and diagnostic processing objectives.
Sensitive vehicle information should be minimized in logs, and IAM permissions should follow least-privilege principles.

## Design Principles
The architecture follows these principles:
- Event-driven processing instead of synchronous end-to-end execution.
- Separation of current DTC state from historical events.
- Idempotent handling of repeated and retried messages.
- Asynchronous AI analysis through SQS.
- Evidence-based diagnostic reasoning through retrieval-augmented generation.
- Independent scaling of ingestion and analysis workloads.
- Low operational overhead for an initial fleet of approximately 100 vehicles.
- Extensibility toward larger fleets, additional diagnostic domains, and predictive maintenance.

## Future Enhancements
Potential future capabilities include:
- Specialist agents for Chassis, Body, and Network diagnostics.
- Coordination and reconciliation of specialist-agent results.
- Fleet-wide DTC trend analysis.
- Time-series telemetry storage.
- Vehicle health dashboards.
- Service, battery, and oil maintenance recommendations.
- Driver notifications that distinguish monitoring from immediate action.
- Guardrails and stronger validation of diagnostic outputs.
- Improved operational dashboards and automated recovery workflows.

## Project Status
The SmartFleet project is being developed incrementally.
The current development focus includes vehicle-event simulation, DTC lifecycle management, event-driven processing, and local integration of the PowertrainAgent with the diagnostic Knowledge Base through MCP.
Cloud deployment and broader fleet integration are planned as subsequent steps.
The architecture is designed to evolve from an initial diagnostic workflow into a multi-domain intelligent fleet-maintenance platform.