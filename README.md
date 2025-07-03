# Lab Automation Notes

Some tips and things of note while learning laboratory automation techniques, especially with Opentrons (@Opentrons):

---

## **How Opentrons Controls Modules**

- **Module Control via Robot Drivers:**  
  Opentrons modules (like the Temperature Module, Magnetic Module, etc.) are controlled through drivers installed on the robot itself (OT-2 or Flex).
- **Direct PC Control Not Supported:** You cannot plug a module directly into your PC and control it. All module commands must be sent to the robot, which then communicates with the modules via its own hardware and firmware.
- **Protocol API:** All module operations (e.g., setting temperature, engaging magnets) are performed by writing Python protocols that use the Opentrons Protocol API. These protocols are uploaded to and executed by the robot.

---

## **Opentrons HTTP API & Network Automation**

- The Opentrons robot (OT-2 or Flex) is essentially a Raspberry Pi running Linux, with the Opentrons software stack and drivers installed.
- You can interact with the robot over your local network using the Opentrons HTTP API. This allows for:
  - Uploading protocols
  - Starting and stopping runs
  - Monitoring run status and logs
  - Controlling modules (temperature, magnetic, etc.)
  - Integrating with external automation scripts, sensors, or LIMS systems

### **Finding the Robot's IP Address**

- The robot's IP address can be found in the Opentrons App under the robot's details, or by checking your network's DHCP client list.
- The default port for the Opentrons HTTP API is `31950`.
- Example usage in scripts:
  ```python
  OT_API = "http://169.254.6.242:31950"
  ```

### **Example: Uploading and Running Protocols**

```python
import requests
import os

protocol_path = "part1.py"
HEADERS = {"opentrons-version": "*"}
OT_API = "http://169.254.6.242:31950"

# Upload protocol
with open(protocol_path, "rb") as f:
    files = {"files": (os.path.basename(protocol_path), f, "text/x-python")}
    resp = requests.post(f"{OT_API}/protocols", files=files, headers=HEADERS)
protocol_id = resp.json()["data"]["id"]

# Create and start run
payload = {"data": {"protocolId": protocol_id}}
resp = requests.post(f"{OT_API}/runs", json=payload, headers=HEADERS)
run_id = resp.json()["data"]["id"]
play_payload = {"data": {"actionType": "play"}}
requests.post(f"{OT_API}/runs/{run_id}/actions", json=play_payload, headers=HEADERS)
```

### **Logging and Automation**

- Use Python's `logging` module to keep detailed logs of your automation process, including protocol uploads, run status, and errors.
- Example logging setup:

  ```python
  import logging
  from datetime import datetime
  import sys

  log_filename = (
      f"yoctopuce_opentrons_automation_{datetime.now().strftime('%Y%m%d_%H%M%S')}.log"
  )
  logging.basicConfig(
      level=logging.INFO,
      format="%(asctime)s [%(levelname)s] %(message)s",
      handlers=[logging.FileHandler(log_filename), logging.StreamHandler(sys.stderr)],
  )
  logger = logging.getLogger(__name__)
  ```

---

## **Protocol Scripting Limitations**

- **No 3rd Party Packages:**  
  You cannot import or use third-party Python packages (e.g., `numpy`, `pandas`, or custom hardware libraries) in Opentrons protocols. Only the built-in Python standard library and the Opentrons API are available. This is for security and reproducibility reasons.

- **No Dynamic Parameters During Runs:**  
  All protocol parameters (such as volumes, temperatures, sample counts) must be set before the run starts.

  - You cannot prompt the user for input or change parameters in the middle of a run.
  - The only way to pause for user intervention is with `protocol.pause()`, which requires the user to manually resume the protocol via the Opentrons App or touchscreen.

- **No Dynamic Protocol Generation During Runs:**
  - The protocol must be fully defined before execution. You cannot dynamically generate new steps or change the protocol structure (e.g., add/remove steps) during a run.
  - You **can** use Python control structures (like `for` loops, `if` statements) to generate a variable number of steps at protocol creation time, _**but the number and nature of steps are fixed once the protocol starts running**_.
  - If you want to change the number of steps based on user input or external data, you must generate a new protocol file and upload it to the robot before starting the run.

---

## **Working Around Protocol Limitations: Dynamic Automation with Script Chaining and Templates**

See the [Protocol Scripting Limitations](#protocol-scripting-limitations) section above for the core restrictions. To work around these limitations:

- **Split your workflow into many small protocol scripts** (each handling a single step or chunk)
- **Use a template engine (like `Jinja2`) to generate scripts dynamically** based on real-time data or feedback
- **Orchestrate the execution of these scripts via an external automation script** (using the HTTP API)

#### **Example: Microadjustment Protocol Template**

A Jinja2 template for a temperature micro-adjustment protocol:

```python
# template_microadjust.py.j2
metadata = {
    "protocolName": "Microadjust Temperature",
    "author": "Automated Script",
    "apiLevel": "2.15"
}

def run(protocol):
    temp_mod = protocol.load_module("temperature module gen2", 3)
    temp_mod.set_temperature(celsius={{ temperature }})
    protocol.delay(seconds={{ delay_seconds }})
    temp_mod.deactivate()
```

#### **Example: Generating and Running Scripts Dynamically**

From your automation script (`yoctopuce_opentrons_automation.py`):

```python
from jinja2 import Template

def generate_microadjust_protocol(temperature, delay_seconds, output_path):
    with open("template_microadjust.py.j2") as f:
        template = Template(f.read())
    protocol_code = template.render(temperature=temperature, delay_seconds=delay_seconds)
    with open(output_path, "w") as f:
        f.write(protocol_code)

# Later, upload and run the generated protocol as shown above
```

#### **Chaining Scripts for Dynamic Workflows**

You can maintain a list of protocol scripts and their associated parameters, and execute them in sequence:

```python
PARTS_AND_TEMPS = [
    ("part1.py", 20),
    ("part2.py", 25),
    # ...
]

for part_script, temp in PARTS_AND_TEMPS:
    if temp is not None:
        stabilize_temperature(temp)
    protocol_id = upload_protocol(part_script)
    run_id = create_and_start_run(protocol_id)
    wait_for_run_complete(run_id)
```

This approach allows you to:

- Make real-time decisions (e.g., based on sensor feedback)
- Dynamically generate protocol steps
- Work around the static nature of Opentrons protocols

So to summarize, for advanced workflows (e.g., dynamic temperature control, feedback from external sensors), using an **external automation script** to monitor conditions, generate/upload new protocol scripts, and orchestrate multiple protocol runs in sequence may be the way to get your ideal protocol working.

---

## **Best Practices**

- **Use parameters for flexibility:** Use protocol parameters to allow some customization at run start (e.g., number of samples, incubation times).
- **Leverage external automation:** For advanced workflows, use an external automation script to monitor, generate, and run protocols as needed.
- **Keep logs:** Maintain logs for troubleshooting and reproducibility.
- **Verify network setup:** Always check the robot's IP and network accessibility before running automation scripts.

---

## More Automation Resources and Projects

- [Opentrons Protocol API Documentation](https://docs.opentrons.com/v2/index.html)
- [Opentrons Advanced Running and Control](https://docs.opentrons.com/v2/new_advanced_running.html)
- [labautomation.io](https://labautomation.io/) (lab automation forums)
- [PyLabRobot](https://github.com/PyLabRobot/pylabrobot) (interactive & hardware agnostic SDK for lab automation)
- [SiLA](https://sila-standard.com/standards/) (Standards to Power the Lab)
- [Opentrons OT-2 SiLA 2 Server](https://github.com/FlorianBauer/ot2-controller) (\*May be outdated due to HTTP API)
