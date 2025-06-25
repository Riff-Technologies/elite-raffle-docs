These are the questions and answers from a session with GLI regarding software architecture and software decisions.

Question: What is the timeframe for certification from GLI?
Answer: RNG certification may take 5-6 weeks, and electronic raffle system certification may take 4 weeks. Time can be reduced by proactively providing system documentation, such as implementation of firewall or networking diagrams.

Question: How can the system self verify that it is running the code it expects?
Answer: The system can pull the checksums of the critical files and display a single checksum on its public website for regulators. The application can validate itself before a critical operation, or it can be validated on a schedule.

Question: What are the critical files?
Answer:
1. The RNG algo and implementation
2. Control program validation (self-test, health checks, self-verification)
3. Critical memory handling (database, e.g. blockchain, memory management for the RSUs, etc.)
4. Generation of ticket validation numbers (unique and unpredictable number that represents the tickets that were sold on an order), e.g. a GUID or UUID. The source code that generates those validation numbers is a critical file.
5. Logic related to the creation and validation of raffles, tickets, draws, etc.

Question: How can we separate the core raffle logic from the business logic?
Answer: Use packages that manage critical data, e.g. Android package that manages raffle data that is a critical file, but the Android app that employs it is not. Payment provider implementation can be kept separate from the core raffle system, but it will have an interface into the system.

Question: Which AWS services might make compliance simpler?
Answer: There's no need to use immutable services - the application code can control what is mutable and what is not, so standard DynamoDB or Postgresql instances should work. A raffle system can take the hashes of its own components and display them on the public website to simplify regulatory checks.

Question: What should we be thinking about early on to avoid future headaches?
Answer: GLI performs two tests: source code review, functionality testing. Some components are difficult to test, so GLI will request documentation for them in lieu of testing, e.g. firewall implementation. Provide documentation to GLI proactively to avoid delays in testing.

Question: Is HSM required for RNG?
Answer: No, cryptographically secure software implementation is good enough. No need to overkill the RNG implementation - it can be a 15 line Javascript function. The RNG team will provide some common algorithms.

Question: Is a VPN client required for the RSU?
Answer: No, it is only required to use HTTPS and standard best practices for security.

Question: We are planning to capture critical event logs from the Raffle Core Service. How searchable do these need to be?
Answer: From a GLI perspective, the logs just need to exist. They can be provided in a text file as proof they are being logged. The lifetime of the logs probably needs to be at least 3 years, but should be configurable by jurisdiction.

Question: How should we implement Lambda function self-verification with the Go runtime?
Answer: The checksum for verification can be added as part of the metadata for the binary when it's uploaded to AWS S3. AWS itself guarantees that that Lambda is using the correct code. So you only need to verify that the checksum of the S3 object matches the expected value, via a separate process. Provide documentation from AWS about how this works.

Question: What happens if the scheduled Verification Lambda detects a checksum mismatch?
Answer: It will not be tested by GLI for certification, but your system should prevent new raffle events from being created, and it should notify the system administrator.

Question: How do we handle reconciliation before initiating a draw?
Answer: Before a draw can be inititiated, the UI interface should prompt the user to confirm that everything has been reconciled. When the user confirms, a critical event should be logged for that user, raffle, and timestamp.  This will be tested by GLI.

Question: What kind of data needs to be stored related to payments?
Answer: GLI only requires that the price for the order and a payment reference be stored related to the order that was created for the tickets, e.g. a transaction ID, a Stripe payment ID, etc. Other payment details can be stored if you wish, but they are not required by GLI.

Question: The Android Package is a certified part of the raffle system. How can it self-verify?
Answer: The Android Package should be a lightweight interface between the Android app and the backend system. It should have minimal logic in it. When the Android APK is built, a checksum can be generated. This checksum should remain constant after certification, and it should be viewable in the Android app. It will not be possible for the package to self-verify. If you need to ship updates to the Android app, you can do so with an over-the-air update that does not affect the APK.
