---
layout: post
title:  "Event-Driven Architecture for Real-Time Communication"
date:   2025-04-02 10:52:00 - 0700
categories: event driven architecture, sqs, sns, aws, firebase, vonage, laravel, ecs
---
## Event-Driven Architecture for Real-Time Communication: Orchestrating the Symphony of Audio/Video Calls

## Introduction

Event-Driven Architecture (EDA) is a software design pattern in which system components communicate and react to events rather than relying on direct requests between services. In this model, an event represents a significant change in state, such as a user clicking a button, a new order being placed, or a sensor detecting a temperature change.

## Key Components of Event-Driven Architecture

* Events – Represent state changes or occurrences (e.g., “OrderPlaced”, “UserLoggedIn”).

* Producers – Generate and publish events when changes occur.

* Event Brokers/Message Queues – Middleware that routes events between producers and consumers (e.g., Kafka, RabbitMQ, AWS SNS/SQS).

* Consumers – Subscribe to and process events asynchronously.

* Event Store (Optional) – A repository to log and replay past events.

## Example Audio/Video Calls Event Driven Architecture

### How it works
![Event Driven Architecture](/swapna/images/EventDrivenArchitecture.png)

The diagram illustrates an Event-Driven Architecture for Audio/Video Calls using AWS SQS, SNS, Firebase, Laravel (Artisan Service), and Vonage APIs. Here's a breakdown of the architecture:
1. User Initiates a Call
A desktop or mobile user initiates an audio/video call request.
The request is sent to the Web Server Service for processing.

2. Processing the Request
The Web Server Service routes the request to the appropriate API Service.
The Chat API Service processes the request and stores relevant details in MySQL.

3. Generate Vonage ID
The Chat API Service interacts with Vonage APIs to create a unique call session ID.
This ID is sent back to the API for further processing.

4. Publishing Call Events to SQS
The system triggers an event to send the incoming call request to AWS SQS.
The event is published under the topic: Chat Service Broadcast (Presence Channel).

5. Queue Processing & Notifications
The Queue Service processes the incoming call request.
Notifications are sent to mobile users via Firebase Cloud Messaging (FCM).
SMS and email alerts are sent using Amazon SNS.

6️. Call Established
Mobile users and desktop users consume the event and receive notifications.
Pusher Webhook and Vonage APIs are used to establish real-time audio/video communication between users.

Key Technologies Used

1. Laravel Artisan Service - Handles background job processing
2. Amazon SQS - Manages event-driven messaging for call requests
3. Amazon SNS - Sends notifications via SMS and email
4. Firebase (FCM) - Sends push notifications to mobile users
5. Vonage API - Manages WebRTC-based audio/video calls
6. Pusher WebSockets - Handles real-time presence and chat broadcasting

Key Benefits of this Event-Driven Approach

* Asynchronous Processing: SQS ensures call events are handled efficiently without blocking users.
* Scalability: The system can handle large volumes of concurrent call requests.
* Decoupled Services: Each service functions independently, improving fault tolerance.
* Real-Time Notifications: Users receive instant call alerts via Firebase, WebSockets, and SMS.

This architecture enables efficient, scalable, and real-time communication between application users and mobile users while leveraging event-driven patterns to manage call states dynamically.

