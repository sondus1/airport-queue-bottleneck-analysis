import random
import pandas as pd
import simpy

# --- 1. SET THE AIRPORT CONFIGURATION ---
NUMBER_OF_LANES = 3     # Total open security lines
SIM_TIME = 480          # Run for 480 minutes (an 8-hour shift)
PAS_ARRIVAL_MEAN = 0.5  # A new passenger arrives every 30 seconds on average
PROCESS_MEAN = 2.0      # Screening takes 2 minutes on average
PROCESS_STD = 0.5       # Bell-curve deviation for screening time

data_logs = []

# --- 2. DEFINE THE PASSENGER EXPERIENCE ---
def passenger_journey(env, passenger_id, security_checkpoint):
    arrival_time = env.now
    
    # Passenger steps in line and requests an open lane resource
    with security_checkpoint.request() as lane_request:
        yield lane_request # Pauses here until a security lane is completely free
        
        # Calculate how long they stood in line
        wait_time = env.now - arrival_time
        
        # Simulate screening duration
        check_duration = max(0.2, random.normalvariate(PROCESS_MEAN, PROCESS_STD))
        yield env.timeout(check_duration) # Hold the lane for this long
        
        departure_time = env.now
        
        # Log data to our internal list
        data_logs.append({
            "Passenger_ID": passenger_id,
            "Arrival_Minute": round(arrival_time, 2),
            "Wait_Time_Minutes": round(wait_time, 2),
            "Check_Duration_Minutes": round(check_duration, 2),
            "Departure_Minute": round(departure_time, 2),
        })

# --- 3. PASSENGER GENERATOR ---
def passenger_spawner(env, security_checkpoint):
    passenger_count = 0
    while True:
        yield env.timeout(random.expovariate(1.0 / PAS_ARRIVAL_MEAN))
        passenger_count += 1
        env.process(passenger_journey(env, f"Passenger_{passenger_count}", security_checkpoint))

# --- 4. EXECUTE THE SIMULATION ---
print("🛫 Running virtual terminal operations...")
random.seed(42)

env = simpy.Environment()
checkpoint_resource = simpy.Resource(env, capacity=NUMBER_OF_LANES)

env.process(passenger_spawner(env, checkpoint_resource))
env.run(until=SIM_TIME)

# --- 5. EXPORT TO CSV ---
df = pd.DataFrame(data_logs)
df.to_csv("airport_baseline_3_lanes.csv", index=False)
print("📊 Success! 'airport_baseline_3_lanes.csv' has been generated.")
