# How to Install Python on Windows

Follow these steps to install Python on a Windows system:

## 1. Download Python
- Go to the official Python website: [https://www.python.org/downloads/](https://www.python.org/downloads/)
- Click on the latest version for Windows and download the installer (e.g., `python-3.x.x.exe`).

## 2. Run the Installer
- Double-click the downloaded installer to start the installation.
- **Important:** Check the box that says **"Add Python to PATH"** at the bottom of the installer window.
- Click **Install Now**.

## 3. Verify Installation
- Open **Command Prompt** (press `Win + R`, type `cmd`, and press Enter).
- Type:
  ```
  python --version
  ```
  or
  ```
  py --version
  ```
- You should see the installed Python version.

## 4. (Optional) Install pip Packages
- To install packages, use:
  ```
  pip install package_name
  ```

## 5. (Optional) Set Up a Virtual Environment
- Navigate to your project folder in Command Prompt.
- Run:
  ```
  python -m venv venv
  venv\Scripts\activate
  ```

---
For more details, visit the [official Python documentation](https://docs.python.org/3/using/windows.html).
