# SmartFleet

SmartFleet AI is an intelligent vehicle diagnostics platform designed to help fleet owners, drivers, and workshop engineers identify vehicle Diagnostic Trouble Codes (DTCs), understand potential root causes, and take appropriate corrective action.

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
flowchart TD
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

### Architecture note:### The diagram represents the intended event-driven flow. DynamoDB Streams and EventBridge Pipes are used to propagate qualifying DTC state changes to the SQS analysis queue. The processor and stream-filtering logic determine which changes qualify for analysis.