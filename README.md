# Project-Work
# Simplified CAN Communication Simulation for SimDaaS Autonomy Pvt Ltd
# Full Solution with Error Checking and Retry Mechanism

import random
import time
import queue

# Simulate a CAN bus using Python's queue
can_bus = queue.Queue()

class SensorNode:
    """
    Simulates a virtual sensor node that generates random data packets.
    Includes a simple checksum for basic error detection.
    """
    
    def __init__(self, name):
        self.name = name

    def generate_data(self):
        """
        Create a random sensor reading and generate a packet with a checksum.
        """
        value = round(random.uniform(0, 100), 2)
        data = f"{self.name}_data: {value}"
        checksum = self.compute_checksum(data)
        return {"data": data, "checksum": checksum}

    def compute_checksum(self, data):
        """
        Compute a basic checksum by summing the ASCII values of the characters.
        """
        return sum(ord(char) for char in data)

    def send_message(self):
        """
        Send a message to the CAN bus, with a chance of random corruption.
        """
        packet = self.generate_data()
        if random.random() < 0.2:
            packet["data"] += "_corrupt"  # Simulate corruption
        can_bus.put(packet)
        print(f"{self.name} sent: {packet['data']}")


class Controller:
    """
    Simulates a central controller that receives and validates messages from sensors.
    Handles message errors and retries if needed.
    """
    
    def __init__(self):
        self.log = []

    def receive_messages(self):
        """
        Process all messages in the CAN bus queue, checking for corruption.
        """
        while not can_bus.empty():
            packet = can_bus.get()
            data, checksum = packet["data"], packet["checksum"]
            
            if self.verify_checksum(data, checksum):
                self.log.append(f" Received valid message: {data}")
            else:
                self.log.append(f" Error detected! Corrupted message: {data}")
                self.retry_message(data)

    def verify_checksum(self, data, expected_checksum):
        """
        Verify whether the received data matches the expected checksum.
        """
        computed_checksum = sum(ord(char) for char in data)
        return computed_checksum == expected_checksum

    def retry_message(self, data):
        """
        Retry sending the corrected message after removing corruption.
        """
        corrected_data = data.replace("_corrupt", "")
        corrected_checksum = sum(ord(char) for char in corrected_data)
        can_bus.put({"data": corrected_data, "checksum": corrected_checksum})
        self.log.append(f" Retrying message: {corrected_data}")


# Initialize sensors and controller
sensors = [
    SensorNode("GPS"),
    SensorNode("LiDAR"),
    SensorNode("Proximity1"),
    SensorNode("Proximity2")
]

controller = Controller()

# Simulate the CAN communication process
for cycle in range(5):
    print(f"\n Cycle {cycle + 1}")
    for sensor in sensors:
        sensor.send_message()
    time.sleep(1)
    controller.receive_messages()

# Output the communication log
print("\n Communication Log:")
for entry in controller.log:
    print(entry)
    
''' CAN Bus Schematic (Conceptual Layout):
 Microcontroller (e.g., ESP32, Arduino, or STM32) as the Central Controller.
 Multiple Sensors: GPS, LiDAR, Proximity sensors.
 CAN Transceivers to handle CAN communication.
 Bus Termination Resistors at both ends of the CAN line.
Connections:
 Sensors → CAN Transceivers → CAN High & CAN Low lines
 Transceivers → Microcontroller UART/SPI pins for data'''

