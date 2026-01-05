# QA-Automation-Engineering-Case-Study

Part 1: Debugging Flaky Test Code (30 minutes)

1. Flakiness Issues Identified

    a) Missing waits after login → race conditions
    b) Dynamic dashboard elements load slowly
    c) .all() collects elements too early
    d) No Playwright fixtures → inconsistent state
    e) 2FA unpredictability for some users
    f) CI/CD has slower network + headless browser differences

2. Root Cause Explanation

    CI/CD pipelines run on slower containers, have higher network delays, and use headless browsers with different rendering behavior.
    Tenant dashboards load data dynamically and inconsistently.
    Because the test uses immediate assertions instead of waits, CI timing differences cause flakiness.

3. Fixes Implemented

    a) Added Playwright expect() waits
    b) Added wait_until="networkidle"
    c) Replaced .all() with locator-based waits
    d) Used stable selectors + text-based waits
    e) Ensured tenant data fully loads
    f) Relied on pytest-playwright page fixture for stable browser context

Corrected Test Code:

import pytest
from playwright.sync_api import expect

def test_user_login(page):
    # Navigate with proper waits
    page.goto("https://app.workflowpro.com/login", wait_until="networkidle")

    # Fill login form
    page.fill("#email", "admin@company1.com")
    page.fill("#password", "password123")
    page.click("#login-btn")

    # Wait for dashboard redirect
    expect(page).to_have_url("https://app.workflowpro.com/dashboard", timeout=10000)

    # Wait for dynamic dashboard content
    welcome = page.locator(".welcome-message")
    expect(welcome).to_be_visible(timeout=10000)


def test_multi_tenant_access(page):
    page.goto("https://app.workflowpro.com/login", wait_until="networkidle")

    page.fill("#email", "user@company2.com")
    page.fill("#password", "password123")
    page.click("#login-btn")

    # Wait for dashboard URL
    expect(page).to_have_url(lambda url: "/dashboard" in url)

    # Wait for project cards to appear
    cards = page.locator(".project-card")
    expect(cards.first).to_be_visible(timeout=15000)

    # Validate tenant-specific data
    for i in range(cards.count()):
        expect(cards.nth(i)).to_contain_text("Company2")

Part 2: Test Framework Design (25 minutes)

1. Framework Structure:
     tests/
     ├── web/ (Playwright UI tests)
     ├── mobile/ (Appium/BrowserStack tests)
     ├── api/ (Backend API tests)
    core/
     ├── base/ (BasePage, BaseAPI, BaseTest)
     ├── drivers/ (Playwright, Appium, BrowserStack)
     └── utils/ (env loader, logger, waits, data reader)
    config/
     ├── tenants/ (company1.yaml, company2.yaml)
     ├── browser_config.yaml
     ├── mobile_config.yaml
     └── environment.yaml
    reports/ (HTML screenshots, logs, videos)
    ci/ (Jenkinsfile, GitHub Actions)

2. Base Components:
BasePage
    Encapsulates common actions (click, fill, waits).
    Ensures stability using Playwright expect() waits.

BaseAPI
    Wrapper for GET/POST calls.
    Handles tokens, base URL, headers.

BaseTest
    Loads environment, tenant config, user roles.
    Initializes driver (local/BrowserStack).

3. Configuration Management:
Multi-Tenant
Each tenant has config:
  base_url:
  admin_user:
  manager_user:
  employee_user:

Browser & Device
  browser: chromium/firefox/webkit
  headless: true
  viewport: 1280x720
  device (for mobile)

Environments
  env: staging | prod
  api_url:
  browserstack_enabled:

Test Data
  JSON/YAML files for user credentials & test datasets.

4. Missing Requirements:
  Do we need Allure? Screenshots/videos on failure?
  How many threads? Parallel by tenant or by browser?
  Is seed data available? Mock users?
  Can 2FA be disabled for automation accounts?
  Required OS/browser versions? Real or virtual devices?
  Token refresh? OAuth flow?
  Run on every PR? Nightly full suite?
  Any special waits required?

Part 3: API + UI Integration Test (35 minutes)

import pytest
import requests
from playwright.sync_api import expect

# FIXTURES

@pytest.fixture
def api_headers():
    return {
        "Authorization": "Bearer TEST_TOKEN_COMPANY1",
        "X-Tenant-ID": "company1"
    }

@pytest.fixture
def project_payload():
    return {
        "name": "Test Project Automation",
        "description": "Created via API",
        "team_members": ["user1", "user2"]
    }

@pytest.fixture
def create_project(api_headers, project_payload):
    response = requests.post(
        "https://api.workflowpro.com/api/v1/projects",
        headers=api_headers,
        json=project_payload,
        timeout=10
    )
    assert response.status_code == 201
    return response.json()  # returns {"id": 123, ...}


# MAIN TEST


def test_project_creation_flow(create_project, page, browser):
    project_id = create_project["id"]
    project_name = create_project["name"]

    # ----------------------------------------------
    # 1. WEB UI VALIDATION (Playwright)
    # ----------------------------------------------
    page.goto("https://company1.workflowpro.com/login", wait_until="networkidle")
    page.fill("#email", "admin@company1.com")
    page.fill("#password", "password123")
    page.click("#login-btn")

    # Wait for dashboard
    projects = page.locator(".project-card")
    expect(projects).to_be_visible(timeout=15000)

    # Check if our API project appears
    matching_project = page.locator(f".project-card:has-text('{project_name}')")
    expect(matching_project).to_be_visible()

    # ----------------------------------------------
    # 2. MOBILE VALIDATION (BrowserStack Mobile Web)
    # ----------------------------------------------
    mobile_capabilities = {
        "browserName": "Chrome",
        "platformName": "Android",
        "deviceName": "Samsung Galaxy S22",
        "bstack:options": {
            "projectName": "WorkflowPro",
            "sessionName": "Mobile Project Check"
        }
    }

    # connect to BrowserStack
    mobile = browser.new_context(**mobile_capabilities)
    mobile_page = mobile.new_page()
    mobile_page.goto("https://company1.workflowpro.com", wait_until="networkidle")

    mobile_page.fill("#email", "admin@company1.com")
    mobile_page.fill("#password", "password123")
    mobile_page.click("#login-btn")

    mobile_match = mobile_page.locator(f"text={project_name}")
    expect(mobile_match).to_be_visible(timeout=20000)

    # ----------------------------------------------
    # 3. TENANT ISOLATION CHECK
    # ----------------------------------------------
    # Login from Company2 tenant
    page.goto("https://company2.workflowpro.com/login", wait_until="networkidle")
    page.fill("#email", "admin@company2.com")
    page.fill("#password", "password123")
    page.click("#login-btn")

    # Ensure Project from Company1 is NOT visible in Company2 dashboard
    forbidden = page.locator(f"text={project_name}")
    expect(forbidden).not_to_be_visible(timeout=10000)

    # Cleanup
    mobile.close()
