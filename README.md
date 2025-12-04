
# RF Test Framework – Developer Documentation

## **1. Introduction**

The **RF Test Framework** is a Python-based testing infrastructure built on **pytest** for validating and benchmarking **NI RFSA/RFSG APIs** exposed through the open-source **nimi-python** repository.

The framework enables:

* Consistent and reusable test patterns
* Automated execution across devices
* Integrated debugging tools
* Structured reporting
* Compatibility with **Mobilize** for distributed automated execution

---

## **2. Project Structure**

```
rf-test-framework/
│
├── Devices/
│   └── <DeviceName>/
│       ├── conftest.py
│       └── Tests/
│           ├── test_basic.py
│           └── test_advanced.py
│
├── Debugging/
│   └── __main__.py
│
├── Utility/
│   └── <helper_modules>.py
│
├── Reports/
│   └── <JUnit XML files>
│
├── conftest.py          # Global configuration
├── config.toml
├── pyproject.toml
├── pytest.ini
└── requirements.txt
```

### **Component Breakdown**

#### **Devices/**

Contains per-device test suites:

* `conftest.py` for device-specific fixtures and hooks
* `Tests/` with:

  * `test_basic.py` — foundational tests
  * `test_advanced.py` — advanced and critical tests

#### **Debugging/**

* CLI debugging entry points (inside `__main__.py`)

#### **Utility/**

* Helper utilities for tests and debugging

#### **Reports/**

* Stores JUnit XML files generated after execution

#### **Configuration Files**

* `config.toml` — runtime configuration
* `pyproject.toml` — project metadata and dependencies
* `pytest.ini` — pytest configuration (markers, logging, CLI options)
* `requirements.txt` — Python dependencies

---

## **3. PyTest Foundation**

### **3.1 Fixtures**

Fixtures provide reusable:

* Setup/teardown logic
* Session or resource creation
* Parameter injection

Defined with:

```python
@pytest.fixture
```

**Fixture scopes:**

* function
* class
* module
* session

Fixtures in `conftest.py` automatically apply to all tests inside that directory.

---

### **3.2 Hooks**

Hooks customize pytest’s behavior during:

* Test collection
* Setup
* Execution
* Reporting

Common hooks used:

* `pytest_sessionstart(session)`
* `pytest_sessionfinish(session, exitstatus)`
* `pytest_runtest_call(item)`

Used for:

* Parametrization
* Execution ordering
* Debugging
* CLI extensions

---

### **3.3 Plugins**

Plugins extend functionality:

* `pytest-xdist` — parallel execution
* `pytest-benchmark` — performance benchmarking
* `pytest-cov` — coverage (optional)

---

## **4. Vision & Goals**

The framework aims to:

* Provide a standardized, simple way to write RFSA/RFSG API tests
* Maintain scalable and maintainable test suites
* Enable debugging, reusability, and automation
* Support CI/CD friendly execution
* Maintain clean test logic using pytest fixtures, hooks, and markers

---

## **5. Execution Flow Overview**

1. **Test triggers from:**

   * CLI
   * Mobilize
   * Dashboard

2. **Framework loads:**

   * Configuration
   * Fixtures
   * Parametrization maps
   * Device-specific setup

3. **Test Execution**

   * Runs `test_basic.py` then `test_advanced.py` using ordering rules

4. **Reporting**

   * Generates JUnit XML

5. **Dashboard**

   * Parses messages from JUnit XML

---

## **6. Mobilize Integration**

### **Execution Steps**

* Create/update mako plan (e.g., `rf-test-framework_Mkholiya.yml.mako`)
* Commit and trigger pipeline
* After build finishes, copy generated package name/version
* Configure a Dashboard event:

  * Set package name
  * Enable event
  * Select test suite
  * Configure initiator and target machines
  * Provide mako variables
* Enqueue the event
* Mobilize runs the tests and generates a Mobilize report

---

## **7. Reporting**

### **JUnit XML Reporting**

The framework uses:

```bash
pytest --junitxml=Reports/results.xml
```

Dashboard reads only the **message attribute**.

To format multi-line messages, embed:

```
&#10;
```

---

## **8. Features of RF-Test-Framework**

### **8.1 Test Selection & Categorization**

Use markers to group tests:

```python
@pytest.mark.critical
```

List markers in `pytest.ini`.

Run selected tests:

```bash
pytest -m "critical"
```

---

### **8.2 Debugging Capabilities**

#### **VS Code Debugging**

Example `launch.json`:

```json
{
  "name": "PyTest Debug",
  "type": "python",
  "request": "launch",
  "module": "pytest",
  "args": ["-k", "test_name"]
}
```

---

### **8.3 Parametrization**

* Values stored in `PARAM_MAP`
* `pytest_generate_tests` hook injects parameters dynamically
* Accessed through fixtures

Example:
Inside conftest.py file
```python
PARAM_MAP = {
    "test_wait_until_settled": {
        "frequency": [1e9, 2e9, 3e9],
        "power_level": [-10.0, -5.0],
    },
    "test_attribute_set_float_attribute": {
        "power_level": [round(x, 2) for x in np.arange(-10.0, 5.0, 1.3)],
    },
    "test_attribute_set_power_level":{
        "power_level": [-30.0, -20.0, -10.0, 0.0],
    }
}
```
Inside Test File
```python
@pytest.mark.critical
    @pytest.mark.attribute
    def test_attribute_set_float_attribute(self,power_level=None):
        if use_simulated_session:
            with nirfsg.Session(simulated_5841_name, options=f"Simulate=1, DriverSetup=Model:{simulated_5841_model}") as rfsg_device_session:
                rfsg_device_session.power_level = -10.0 if not power_level else power_level
                assert rfsg_device_session.power_level == -10.0 if not power_level else power_level
        else:
            with nirfsg.Session(real_hw_resource_name) as rfsg_device_session:
                rfsg_device_session.power_level = -10.0 if not power_level else power_level
                assert rfsg_device_session.power_level == -10.0 if not power_level else power_level
```
---

### **8.4 Execution Order**

Control using:

* File naming
* `pytest-order` plugin
* Ordering markers:

```python
@pytest.mark.order(after="test_login")
```

Guarantee:

* `test_basic.py` runs **before** `test_advanced.py`

---

### **8.5 Critical Test Handling**

If a critical test fails, stop executing remaining tests.

```python
@pytest.mark.critical
def test_something():
    ...
```

---

### **8.6 API Usage Search**

Search which tests use a function/API:

```bash
python -m debugging --find-usage "configure_power_level"
```

Outputs:

* Test names
* Line numbers
* Type (method or attribute)

---

### **8.7 Rerun Failed Tests**

Rerun only failed or skipped tests from a Dashboard run:

```bash
python -m debugging --rerun-id <run_id> --category failed
```

---

## **9. Requirements**

### **Python**

Listed in:

```bash
requirements.txt
```

### **NI Software**

* NI-RFSG drivers
* NI MAX runtime configuration

---

## **10. Conclusion**

The RF Test Framework provides a structured, scalable, and automation-ready solution for RF device testing.
Built on top of PyTest, it supports parametrization, debugging utilities, execution ordering, reporting, and distributed automation — enabling efficient RF workflow validation across devices and environments.
