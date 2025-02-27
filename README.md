# Project-Work
# Simplified CAN Communication Simulation for SimDaaS Autonomy Pvt Ltd

1. Install Dependencies

This project only uses Python's standard libraries, so no extra dependencies are needed. However, if you want to create a requirements.txt for reference:


To install dependencies, run:

pip install -r requirements.txt


2. Python Code

A. src/message_bus.py (Simulated CAN Bus)

Handles message transmission between sensors and the controller using a queue.

import queue

class MessageBus:
    def init(self):
        self.bus = queue.Queue()  

    def send(self, message):
        self.bus.put(message)  

    def receive(self):
        if not self.bus.empty():
            return self.bus.get() 
        return None


B. src/sensor_node.py (Sensor Nodes)

Defines different sensor nodes that generate and send messages.

import random
import time

class SensorNode:
    def init(self, name, data_func, bus):
        self.name = name
        self.data_func = data_func  
        self.bus = bus  

    def generate_message(self):
        data = self.data_func()
        checksum = sum(ord(c) for c in data) % 256  
        return f"{self.name}:{data}:{checksum}"

    def send(self):
        message = self.generate_message()
        
        # Simulate a 10% chance of corruption
        if random.random() < 0.1:
            message = message[:-1] + chr(random.randint(0, 255)) 

        self.bus.send(message)


C. src/central_controller.py (Controller)

Handles message verification, detects errors, and requests retries.

import time

class CentralController:
    def init(self, bus):
        self.bus = bus  

    def check_message(self, message):
        try:
            node, data, received_checksum = message.rsplit(":", 2)
            received_checksum = int(received_checksum)
            computed_checksum = sum(ord(c) for c in data) % 256 
            
            if received_checksum != computed_checksum:
                print(f"Error detected in message from {node}. Requesting retry...")
                return False
            print(f"Received valid message from {node}: {data}")
            return True
        except ValueError:
            print("Malformed message received. Ignoring.")
            return False

    def listen(self):
        while True:
            message = self.bus.receive()
            if message:
                if not self.check_message(message):  
                    print("Requesting resend...")
            time.sleep(1)


D. main.py (Runs the Simulation)

Initializes sensors, the controller, and runs the program.

import threading
import time
from src.sensor_node import SensorNode
from src.central_controller import CentralController
from src.message_bus import MessageBus

# Sensor data functions
def gps_data():
    return f"GPS({random.uniform(-90, 90):.2f},{random.uniform(-180, 180):.2f})"

def lidar_data():
    return f"LIDAR({random.randint(1, 100)}m)"

def proximity_data():
    return f"PROX({random.choice(['Safe', 'Warning', 'Danger'])})"

# Initialize message bus (simulated CAN bus)
bus = MessageBus()
# Create sensor nodes
sensors = [
    SensorNode("GPS", gps_data, bus),
    SensorNode("LiDAR", lidar_data, bus),
    SensorNode("Prox1", proximity_data, bus),
    SensorNode("Prox2", proximity_data, bus)
]

# Initialize and start the central controller
controller = CentralController(bus)
controller_thread = threading.Thread(target=controller.listen, daemon=True)
controller_thread.start()

# Start sensors (simulate periodic message sending)
while True:
    for sensor in sensors:
        sensor.send()
        time.sleep(0.5)
