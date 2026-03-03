# 1. Introduction to Object Oriented Design

Are we focusing on high-level class diagram, code structure, or a full implementation?

Different types Object-Oriented Design (OOD) interviews may require varying levels of detail in the design. Here are some common approaches:

- UML Diagrams: A UML class diagram helps illustrate the relationships between classes, including their attributes, methods, and interactions.

- Code Skeleton: In this style, you define the structure of your design directly in code using appropriate class and method declarations, while leaving method bodies unimplemented.

- Working Code: With the renewed emphasis on OOD interviews, interviewers sometimes request fully functional, bug-free implementations. They may also ask for test cases.

Framework for the OOD Interview:

- Step 1: Requirements Gathering
- Step 2: Identify Core Objects
- Step 3: Define Class Responsibilities
  - Start by designing the classes using either a top-down or bottom-up approach: 
    - Top-down Approach: First, identify high-level components or parent classes, then refine their attributes and methods.
    - Bottom-up Approach: Define concrete classes first (attributes, methods) and build relationships from there.
- Step 4: Deep Dive Topics After validating your design with key use cases, refine it to handle edge cases and resolve any inconsistencies.

For Example - Design a Parking Lot System

1. Requirements Gathering (Clarify the problem statement and gather requirements):
   - The system will support parking and unparking vehicles, track space availability, and calculate fees based on vehicle type and parking duration. It should also support three types of vehicles.
   - a bus with a reservation entering the lot. Some spaces are too small or reserved for other types. The system needs to find the most suitable available space while optimizing future availability.

2. Identify Core Objects:
   - When a car enters, the system will find an available space of the appropriate size, assign it, generate a ticket, and mark the space as occupied.
   - When a car leaves, the system will calculate the parking fee based on the duration and vehicle type, update the space status to available, and process payment.
   - If no spaces are available, the system should return an appropriate message. I’ll refine this logic once I have the complete design.
   - If time permits, I’ll extend the design to support configurable pricing, perhaps using a strategy pattern.
   - Core Objects: ParkingLot, ParkingSpace, Vehicle, Ticket, PaymentProcessor

Step 3: Class Design and Code

Defining the Classes

- Starts with foundational components that form the system’s backbone. In the parking lot example, she focuses on: ParkingLot, ParkingSpot, Vehicle, and Ticket.
- The key entities are ParkingLot, ParkingSpot, Vehicle, and Ticket. Each space has attributes like size and availability, and each vehicle has a type. A Ticket will track the entry time and calculate the fee.

```text
ParkingLot
   |                
   |                
ParkingSpot <------- Ticket
    |                  |
    |                  |
Vehicle <---------------

- ParkingLot contains multiple ParkingSpots.
- Each ParkingSpot can hold one Vehicle.
- A Ticket links a Vehicle to a ParkingSpot and tracks the time.
```
Ensures that each class is well-defined and adheres to OOP principles like encapsulation, single responsibility, and inheritance:

We can define a base Vehicle class with subclasses like Car, Motorcycle, and Bus since their parking requirements and fee calculations differ.


- Code Implementation

  - ParkingLot: manages a collection of spots and handles assignments.
  - ParkingSpot: tracks size, availability, and assigned vehicle.
  - Ticket: stores entry time and calculates the fee.


Step 4: Deep Dive Topics

Addressing Gaps

- For edge cases, I’d add logic to handle full lots and group spaces for buses. I’d also include validation for invalid tickets during checkout.


Summarizing the Design

 his design supports key use cases, scales to different vehicle types, and includes logic for core edge cases. If time allows, I’d explore enhancements like dynamic pricing based on time of day.


What If the Interview Doesn’t Go as Planned?

- I’ll start with a high-level overview, then we can dive deeper where needed.
  
- Ask the interviewer what level of depth they’d like you to go into:
“Would you prefer a high-level architecture here or a more detailed class breakdown?”

- Would you mind giving a specific case where you see this approach falling short?”

- Would it be helpful if I clarified anything or focused on a specific part of the design?
  

- Addressing Concurrency in OOD Interviews





