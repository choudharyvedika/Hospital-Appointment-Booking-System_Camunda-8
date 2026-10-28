# Hospital Appointment Booking System (Camunda 8)

An end-to-end, automated hospital appointment scheduling engine powered by **Camunda 8 (Zeebe)**. The solution integrates DMN decision tables for real-time capacity checks, user task workflows for staff booking, reusable HTTP microservice workers for patient messaging, and an interrupting event subprocess for immediate cancellations.

---

## Process Overview

The workflow coordinates three primary stages:
1. **Rule Evaluation (DMN)**: Inspects the incoming department and current booking load against department limits.
2. **Booking & Staff Handling (BPMN)**: Directs available appointments to staff review while rejecting requests when capacity is exceeded.
3. **Automated Telegram Notification (Call Activity)**: Executes an HTTP POST task to notify patients of confirmations, rejections, or cancellations.

---

## Architecture & Workflows

### 1. Main Booking Workflow (`booking.bpmn`)
* **Process ID**: `hospital-booking`
* **Trigger**: Starts on receipt of a booking request payload containing:
  * `patientId` (String)
  * `patientName` (String)
  * `department` (String)
  * `bookedCount` (Number)
  * `preferredSlot` (String)
* **Check slot**: A Business Rule Task calling DMN table `slot_decision`. Outputs `slotStatus` and `suggestedAction`.
* **Branching Gateway (`Slot Available?`)**:
  * **`AVAILABLE`**: Routes to User Task `Book slot` (assigned to `demo`), then triggers the `send-notification` Call Activity with a confirmation message.
  * **`FULL`**: Routes directly to `send-notification` Call Activity with a rejection message informing the patient that the department is full.
* **Interrupting Event Subprocess (`Handle cancellation`)**:
  * Catches `Message_CancelBooking` correlated by `patientId`.
  * Calls Service Task (`release-slot-service`) to free the allocated slot.
  * Dispatches a cancellation confirmation notification via Call Activity and terminates with `Booking Cancelled`.

### 2. Slot Capacity Rules (`slot_decision.dmn`)
* **Decision ID**: `slot_decision`
* **Hit Policy**: `UNIQUE`
* **Rules**:
  * `department == "Cardiology"` AND `bookedCount < 10` $\rightarrow$ `slotStatus = "AVAILABLE"`, `suggestedAction = "PROCEED_BOOKING"`
  * `department == "Cardiology"` AND `bookedCount >= 10` $\rightarrow$ `slotStatus = "FULL"`, `suggestedAction = "REJECT_SLOT_FULL"`

### 3. Messaging Service (`notification.bpmn`)
* **Process ID**: `send-notification`
* **Type**: Modular Zeebe HTTP Worker (`type: http`)
* **Endpoint**: Dispatches an HTTP `POST` call to the notification gateway (`https://httpbin.org/post`) with payload:
  ```json
  {
    "chat_id": "=recipientId",
    "text": "=messageText"
  }

  FILE STRUCTURE:
  .
├── bpmn/
│   ├── booking.bpmn          # Main booking workflow & cancellation subprocess
│   └── notification.bpmn     # Reusable Telegram notification microservice
├── dmn/
│   └── slot_decision.dmn     # DMN decision table for slot availability
└── README.md

Deployment & Execution
Prerequisites
Camunda 8 (SaaS cluster or Self-Managed local instance via Docker Compose)

Camunda Desktop Modeler or zbctl CLI

Deploying Artifacts
Deploy all BPMN and DMN models to your Zeebe cluster in the following order:
# 1. Deploy the DMN decision table
zbctl deploy dmn/slot_decision.dmn

# 2. Deploy the reusable notification sub-process
zbctl deploy bpmn/notification.bpmn

# 3. Deploy the primary booking workflow
zbctl deploy bpmn/booking.bpmn

Testing Scenarios
Scenario A: Successful Booking
Input Payload:
{
  "patientId": "PAT_101",
  "patientName": "John Doe",
  "department": "Cardiology",
  "bookedCount": 3,
  "preferredSlot": "10:00 AM"
}
Expected Result: DMN evaluates to AVAILABLE $\rightarrow$ creates a task for user demo in Tasklist $\rightarrow$ staff completes task $\rightarrow$ Telegram confirmation sent $\rightarrow$ process completes with status Booking Completed.

Scenario B: Capacity Exceeded
Input Payload:
{
  "patientId": "PAT_102",
  "patientName": "Jane Smith",
  "department": "Cardiology",
  "bookedCount": 10,
  "preferredSlot": "11:30 AM"
}
Expected Result: DMN evaluates to FULL $\rightarrow$ skips user task $\rightarrow$ Telegram slot-full notification sent $\rightarrow$ process terminates with status Booking Rejected.

Scenario C: Mid-flight Cancellation
Publish the following message while the instance is waiting at Book slot:

Message Name: Message_CancelBooking

Correlation Key: PAT_101
zbctl publish message "Message_CancelBooking" --correlationKey "PAT_101"
zbctl publish message "Message_CancelBooking" --correlationKey "PAT_101"
Expected Result: Active user task is interrupted $\rightarrow$ release-slot-service job is created $\rightarrow$ cancellation message is dispatched $\rightarrow$ process ends at Booking Cancelled.
