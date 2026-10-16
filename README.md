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