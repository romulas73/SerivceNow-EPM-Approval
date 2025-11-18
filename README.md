# ServiceNow RITM Approval Integration
## Technical Documentation

---

## Table of Contents
1. [Overview](#overview)
2. [Architecture](#architecture)
3. [Components](#components)
4. [Prerequisites](#prerequisites)
5. [Installation Steps](#installation-steps)
6. [Configuration](#configuration)
7. [Testing](#testing)
8. [Troubleshooting](#troubleshooting)
9. [Appendix](#appendix)

---

## Overview

### Purpose
This integration provides a custom approval interface for Request Items (RITMs) in ServiceNow, specifically designed for the "EPM Elevation Request" catalog item. It enables approvers to review detailed request information, approve or reject requests with a specified duration, and automatically transmit approved request data to an external system via REST API.

### Key Features
- Custom approval UI with two-column layout displaying all request variables
- Flexible approval duration configuration with three types:
  - **Hours**: 1-24 hours (default 24)
  - **Days**: 1-30 days (default 30)
  - **Months**: 1-12 months (default 12)
- Dynamic form validation based on approval type
- Automatic state management (Closed Complete/Closed Incomplete)
- REST API integration for approved requests
- Restricted to specific catalog item ("EPM Elevation Request")

### Business Flow
1. User submits an EPM Elevation Request through the Service Catalog
2. Approver opens the RITM and clicks "Display Approval Form" button
3. Approver reviews all request details in the custom two-column form
4. Approver either:
   - **Approves**: Selects approval type (Hours/Days/Months) and duration, closes RITM as Complete
   - **Rejects**: Closes RITM as Incomplete
5. On approval, system automatically sends all RITM data including approval details to external API
6. Parent request is closed when all items are complete

---

## Architecture

### Component Diagram

```
┌─────────────────┐
│   RITM Form     │
│  (sc_req_item)  │
└────────┬────────┘
         │
         │ Click Button
         ▼
┌─────────────────┐
│   UI Action     │
│ (Client-side)   │
└────────┬────────┘
         │
         │ Opens Modal
         ▼
┌─────────────────┐
│    UI Page      │
│   (Jelly/HTML)  │
└────────┬────────┘
         │
         │ Calls AJAX
         ▼
┌─────────────────┐
│ Script Include  │
│(RITMApprovalUtil)│
└────────┬────────┘
         │
         │ Updates RITM
         ▼
┌─────────────────┐
│ Business Rule   │
│   (Async)       │
└────────┬────────┘
         │
         │ Sends Data
         ▼
┌─────────────────┐
│  REST Message   │
│ (External API)  │
└─────────────────┘
```

### Data Flow

1. **Retrieve Data**: UI Page → Script Include → RITM Variables
2. **User Decision**: UI Page → Client Script → Script Include
3. **Update RITM**: Script Include → RITM Record (state, variables)
4. **Trigger Integration**: RITM Update → Business Rule → REST API
5. **External Sync**: REST Message → External System

---

## Components

### 1. UI Action
**Name**: Display Approval Form  
**Table**: sc_req_item  
**Type**: Form Button (Client-side)  
**Purpose**: Displays the custom approval form button on RITM records

### 2. UI Page
**Name**: ritm_approval_form_page  
**Type**: Jelly/HTML UI Page  
**Purpose**: Renders the approval form with all RITM variables in a two-column layout

### 3. Client Script
**Name**: RITM Approval Form Script  
**Type**: UI Page Client Script  
**Purpose**: Handles approve/reject button clicks and AJAX communication

### 4. Script Include
**Name**: RITMApprovalUtil  
**Type**: Server-side Script Include (Client Callable)  
**Purpose**: 
- Retrieves RITM variable data
- Processes approval/rejection logic
- Updates RITM state and variables

### 5. Business Rule
**Name**: Send RITM Data on Approval  
**Table**: sc_req_item  
**Type**: Async Business Rule  
**Purpose**: Automatically sends RITM data to external API when approved

### 6. REST Message
**Name**: Send RITM Data  
**Type**: Outbound REST Message  
**Purpose**: Defines the REST API endpoint and authentication for external integration

---

## Prerequisites

### Required Variables on Catalog Item
The following variables must exist on the "EPM Elevation Request" catalog item:

| Variable Name | Type | Description |
|--------------|------|-------------|
| computer_name | String | Name of the computer |
| display_name | String | Display name of the application |
| product_version | String | Version of the product |
| product_name | String | Name of the product |
| file_package | String | File package identifier |
| package_name | String | Name of the package |
| username | String | Username requesting elevation |
| publisher | String | Software publisher |
| original_file_name | String | Original filename |
| file_name | String | Current filename |
| source_type | String | Source type of the request |
| source_name | String | Source name |
| file_location | String | Location of the file |
| file_size | String | Size of the file |
| access_type | String | Type of access requested |
| access_target_type | String | Target type for access |
| agent_id | String | Agent identifier |
| checksum | String | File checksum |
| justification | String (Multi-line) | Business justification |
| approved_type | Choice/String | Type of approval duration (hour, day, month) |
| approved_length | Integer | Length of approval (auto-populated) |

### ServiceNow Permissions
- Admin role or equivalent permissions to create:
  - UI Actions
  - UI Pages
  - Client Scripts
  - Script Includes
  - Business Rules
  - REST Messages

### External API Requirements
- REST API endpoint that accepts POST requests
- Authentication credentials (Basic, OAuth2, or API Key)
- Ability to receive JSON payload

---

## Installation Steps

### Step 1: Create Script Include

1. Navigate to **System Definition > Script Includes**
2. Click **New**
3. Fill in the following:
   - **Name**: `RITMApprovalUtil`
   - **API Name**: `RITMApprovalUtil`
   - **Client callable**: ✓ **Checked**
   - **Active**: ✓ **Checked**
   - **Accessible from**: All application scopes
4. Paste the Script Include code (see Appendix A)
5. Click **Submit**

### Step 2: Create UI Page

1. Navigate to **System UI > UI Pages**
2. Click **New**
3. Fill in the following:
   - **Name**: `ritm_approval_form_page`
   - **Category**: General
4. In the **HTML** field, paste the UI Page code (see Appendix B)
5. Leave Client Script and Processing Script empty
6. Click **Submit**

### Step 3: Create Client Script

1. Navigate to **System Definition > Client Scripts**
2. Click **New**
3. Fill in the following:
   - **Name**: `RITM Approval Form Script`
   - **UI Type**: All
   - **Type**: onLoad (or leave as default)
   - **UI Page**: `ritm_approval_form_page`
   - **Active**: ✓ **Checked**
4. Paste the Client Script code (see Appendix C)
5. Click **Submit**

### Step 4: Create UI Action

1. Navigate to **System Definition > UI Actions**
2. Click **New**
3. Fill in the following:
   - **Name**: `Display Approval Form`
   - **Table**: `sc_req_item`
   - **Action name**: `display_approval_form`
   - **Form button**: ✓ **Checked**
   - **Client**: ✓ **Checked**
   - **Active**: ✓ **Checked**
   - **Order**: 100 (or your preference)
   - **Condition**: 
     ```javascript
     current.state != 3 && current.state != 4 && current.cat_item.name == 'EPM Elevation Request'
     ```
4. In the **Script** field, paste the UI Action code (see Appendix D)
5. Click **Submit**

### Step 5: Create REST Message

1. Navigate to **System Web Services > Outbound > REST Message**
2. Click **New**
3. Fill in the following:
   - **Name**: `Send RITM Data`
   - **Endpoint**: Your external API endpoint (e.g., `https://api.example.com/ritm`)
   - **Authentication type**: Select appropriate type (Basic, OAuth2, etc.)
   - Configure authentication credentials as needed
4. Click **Submit**

### Step 6: Create REST Message HTTP Method

1. Open the REST Message you just created
2. In the **HTTP Methods** related list, click **New**
3. Fill in the following:
   - **Name**: `POST`
   - **HTTP method**: `POST`
   - **Endpoint**: Use parent endpoint or specify different
4. Click **Submit**

### Step 7: Create Business Rule

1. Navigate to **System Definition > Business Rules**
2. Click **New**
3. Fill in the following:
   - **Name**: `Send RITM Data on Approval`
   - **Table**: `sc_req_item`
   - **Active**: ✓ **Checked**
   - **Advanced**: ✓ **Checked**
   - **When**: async
   - **Insert**: ☐ Unchecked
   - **Update**: ✓ **Checked**
   - **Filter Conditions**: 
     - State changes to Closed Complete
     - Or use condition: `current.state.changes() && current.state == 3`
4. In the **Advanced** tab, paste the Business Rule code (see Appendix E)
5. Click **Submit**

---

## Configuration

### UI Action Customization

**Adjust Catalog Item Name**:
If your catalog item has a different name, update the condition:
```javascript
current.state != 3 && current.state != 4 && current.cat_item.name == 'YOUR_CATALOG_ITEM_NAME'
```

**Change Button Label**:
Modify the UI Action's **Action label** field in the form.

### Approval Duration

**Change Default Values**:
In the UI Page script section, locate:
```javascript
window.onload = function() {
    document.getElementById('approved_type').value = 'month';
    document.getElementById('approved_length').value = '12';
};
```
Change the values to your desired defaults.

**Change Maximum Limits**:
In the `updateApprovalLimits()` function, modify:
```javascript
if (selectedType === 'hour') {
    lengthInput.max = 24;  // Change this value
    lengthInput.value = 24;
} else if (selectedType === 'day') {
    lengthInput.max = 30;  // Change this value
    lengthInput.value = 30;
} else if (selectedType === 'month') {
    lengthInput.max = 12;  // Change this value
    lengthInput.value = 12;
}
```

**Add/Remove Approval Types**:
In the UI Page HTML, modify the select options:
```html
<select id="approved_type" onchange="updateApprovalLimits()">
    <option value="">-- Select Type --</option>
    <option value="hour">Hours</option>
    <option value="day">Days</option>
    <option value="month">Months</option>
    <!-- Add new types as needed -->
</select>
```

### REST API Configuration

**Update Endpoint**:
1. Navigate to the REST Message record
2. Update the **Endpoint** field with your API URL

**Configure Authentication**:
1. Select appropriate **Authentication type**
2. Fill in credentials:
   - **Basic**: Username and Password
   - **OAuth 2.0**: Client ID, Client Secret, Token URL
   - **Custom**: Configure headers as needed

**Add Custom Headers**:
In the HTTP Method record, add HTTP Headers:
- Click **New** in the HTTP Request Headers related list
- Add headers like `x-api-key`, custom authentication, etc.

### Error Handling

**Business Rule Retry Logic**:
To add retry logic, wrap the REST call in a try-catch and use a scheduled job for retries:

```javascript
try {
    var response = request.execute();
    if (response.getStatusCode() >= 400) {
        // Log error and schedule retry
        gs.error('REST call failed, scheduling retry');
    }
} catch (ex) {
    gs.error('Exception: ' + ex.message);
}
```

---

## Testing

### Unit Testing

**Test 1: UI Action Visibility**
1. Navigate to any RITM for "EPM Elevation Request"
2. Verify "Display Approval Form" button appears
3. Navigate to RITM for different catalog item
4. Verify button does NOT appear

**Test 2: Approval Form Display**
1. Open RITM for "EPM Elevation Request"
2. Click "Display Approval Form"
3. Verify all variables display correctly in two columns
4. Verify justification appears as full-width read-only text area
5. Verify "Approval Type" dropdown shows Hours, Days, Months
6. Verify "Approval Length" defaults to 12 (when Months is selected)
7. Change approval type and verify max values update:
   - Hours: max 24
   - Days: max 30
   - Months: max 12

**Test 3: Approval Workflow**
1. Open approval form
2. Select approval type (e.g., "Days")
3. Set length to 15
4. Click "Approve"
5. Verify success message shows "15 days"
6. Verify RITM state changes to "Closed Complete"
7. Verify `approved_type` variable is set to "day"
8. Verify `approved_length` variable is set to "15"
9. Check System Logs for successful variable updates

**Test 4: Rejection Workflow**
1. Open approval form
2. Click "Reject"
3. Confirm rejection
4. Verify RITM state changes to "Closed Incomplete"
5. Verify no REST call is made (check logs)

**Test 5: REST Integration**
1. Approve a request with 12 months
2. Check System Logs (**System Logs > System Log > All**)
3. Verify log entries:
   - "processApproval called - RITM: ..., Type: month, Length: 12"
   - "Updated variable approved_type to value: month"
   - "Updated variable approved_length to value: 12"
   - "REST API Response Status: 200"
4. Check external system for received data
5. Verify payload includes:
   - `approved_type: "month"`
   - `approved_length: "12"`
6. Verify all other variables are present in payload

### Integration Testing

**End-to-End Test**:
1. Submit new "EPM Elevation Request" via Service Portal
2. Approver receives notification (if configured)
3. Approver opens RITM and clicks "Display Approval Form"
4. Approver reviews all details in two-column layout
5. Approver selects "Hours" and enters "8"
6. Approver clicks "Approve"
7. Verify RITM closes as Complete with note "Approved for 8 hours by [username]"
8. Verify REST API receives data with `approved_type: "hour"` and `approved_length: "8"`
9. Verify external system processes data correctly
10. Verify parent request closes if all items complete

---

## Troubleshooting

### Issue: Button Not Showing

**Possible Causes**:
- Catalog item name doesn't match condition
- RITM already closed
- UI Action not active

**Resolution**:
1. Check catalog item name exactly matches condition
2. Verify RITM state is not 3 or 4
3. Navigate to UI Action record and verify Active = true
4. Check user has sufficient permissions

### Issue: Form Not Loading Variables

**Possible Causes**:
- Script Include not client callable
- Variables don't exist on catalog item
- Jelly syntax error

**Resolution**:
1. Verify Script Include has "Client callable" checked
2. Navigate to catalog item and verify all variables exist
3. Check UI Page for syntax errors
4. Check browser console for JavaScript errors

### Issue: Approved Type/Length Not Saving

**Possible Causes**:
- Variables don't exist on catalog item
- Script Include not updating correctly
- Business Rule executing before variable save

**Resolution**:
1. Verify catalog item has `approved_type` and `approved_length` variables
2. Check System Logs for these entries:
   - "processApproval called - RITM: ..., Type: ..., Length: ..."
   - "Updated variable approved_type to value: ..."
   - "Updated variable approved_length to value: ..."
3. If "Variable definition not found" appears, create the variables on catalog item
4. Verify Business Rule is set to **async** (not "after")
5. Check browser console for JavaScript errors during approval
6. Verify Script Include has correct function signature: `_approveRITM: function(ritm, type, length)`

### Issue: REST Call Failing

**Possible Causes**:
- Incorrect endpoint URL
- Authentication failure
- Network connectivity issues
- External API down

**Resolution**:
1. Check System Logs for detailed error message
2. Verify endpoint URL is correct and reachable
3. Test REST Message manually:
   - Navigate to REST Message
   - Click "Test" under HTTP Methods
4. Verify authentication credentials
5. Check external API logs
6. Use REST API Explorer to debug

### Issue: Business Rule Not Firing

**Possible Causes**:
- Condition not met
- Business Rule not active
- Business Rule order issue

**Resolution**:
1. Check Business Rule conditions
2. Verify Active = true
3. Check "When" is set to "async"
4. Review filter conditions
5. Add debug logging at start of business rule:
   ```javascript
   gs.info('Business Rule fired for RITM: ' + current.number);
   ```

### Debug Logging

**Enable Debug Logging**:

Add to Script Include methods:
```javascript
gs.info('processApproval - Type: ' + approvedType + ', Length: ' + approvedLength);
gs.info('_setVariableValue - Setting ' + variableName + ' to: ' + value);
```

Add to Business Rule:
```javascript
gs.info('Approved Type from variable: ' + payload.variables.approved_type);
gs.info('Approved Length from variable: ' + payload.variables.approved_length);
gs.info('Full Payload: ' + JSON.stringify(payload));
```

Add to Client Script (browser console):
```javascript
console.log('Approval Type selected: ' + type);
console.log('Approval Length entered: ' + length);
```

View logs:
1. Navigate to **System Logs > System Log > All**
2. Filter by source or message
3. For browser logs, open Developer Tools (F12) and check Console tab

---

## Appendix

### Appendix A: Script Include Code
```javascript
var RITMApprovalUtil = Class.create();
RITMApprovalUtil.prototype = Object.extendsObject(AbstractAjaxProcessor, {
    
    getRITMData: function(ritmId) {
        var data = {};
        
        if (!ritmId) {
            return data;
        }
        
        var ritm = new GlideRecord('sc_req_item');
        if (ritm.get(ritmId)) {
            data.computer_name = this._getVariableValue(ritm, 'computer_name');
            data.display_name = this._getVariableValue(ritm, 'display_name');
            data.product_version = this._getVariableValue(ritm, 'product_version');
            data.product_name = this._getVariableValue(ritm, 'product_name');
            data.file_package = this._getVariableValue(ritm, 'file_package');
            data.package_name = this._getVariableValue(ritm, 'package_name');
            data.username = this._getVariableValue(ritm, 'username');
            data.publisher = this._getVariableValue(ritm, 'publisher');
            data.original_file_name = this._getVariableValue(ritm, 'original_file_name');
            data.file_name = this._getVariableValue(ritm, 'file_name');
            data.source_type = this._getVariableValue(ritm, 'source_type');
            data.source_name = this._getVariableValue(ritm, 'source_name');
            data.file_location = this._getVariableValue(ritm, 'file_location');
            data.file_size = this._getVariableValue(ritm, 'file_size');
            data.access_type = this._getVariableValue(ritm, 'access_type');
            data.access_target_type = this._getVariableValue(ritm, 'access_target_type');
            data.agent_id = this._getVariableValue(ritm, 'agent_id');
            data.checksum = this._getVariableValue(ritm, 'checksum');
            data.justification = this._getVariableValue(ritm, 'justification');
        }
        
        return data;
    },
    
    processApproval: function() {
        var ritmId = this.getParameter('sysparm_ritm_id');
        var action = this.getParameter('sysparm_action');
        var approvedMonths = this.getParameter('sysparm_approved_months');
        
        if (!ritmId) {
            return 'error: No RITM ID provided';
        }
        
        var ritm = new GlideRecord('sc_req_item');
        if (!ritm.get(ritmId)) {
            return 'error: RITM not found';
        }
        
        try {
            if (action == 'approve') {
                this._approveRITM(ritm, approvedMonths);
            } else if (action == 'reject') {
                this._rejectRITM(ritm);
            } else {
                return 'error: Invalid action';
            }
            
            return 'success';
        } catch (e) {
            gs.error('Error processing RITM approval: ' + e.message);
            return 'error: ' + e.message;
        }
    },
    
    _approveRITM: function(ritm, months) {
        this._setVariableValue(ritm, 'approved_months', months);
        
        ritm.state = 3;
        ritm.stage = 'fulfilled';
        ritm.close_notes = 'Approved for ' + months + ' months by ' + gs.getUserName();
        ritm.closed_at = new GlideDateTime();
        ritm.closed_by = gs.getUserID();
        ritm.work_notes = 'Approved Months: ' + months;
        
        ritm.update();
        
        this._checkAndCloseRequest(ritm.request.toString());
    },
    
    _rejectRITM: function(ritm) {
        ritm.state = 4;
        ritm.stage = 'request_cancelled';
        ritm.close_notes = 'Rejected by ' + gs.getUserName();
        ritm.closed_at = new GlideDateTime();
        ritm.closed_by = gs.getUserID();
        
        ritm.update();
        
        this._checkAndCloseRequest(ritm.request.toString());
    },
    
    _getVariableValue: function(ritm, variableName) {
        var value = '';
        
        var gr = new GlideRecord('sc_item_option_mtom');
        gr.addQuery('request_item', ritm.sys_id);
        gr.query();
        
        while (gr.next()) {
            var option = gr.sc_item_option.getRefRecord();
            if (option.item_option_new.name == variableName) {
                value = gr.sc_item_option.value.toString();
                break;
            }
        }
        
        return value;
    },
    
    _setVariableValue: function(ritm, variableName, value) {
        var gr = new GlideRecord('sc_item_option_mtom');
        gr.addQuery('request_item', ritm.sys_id);
        gr.query();
        
        var found = false;
        while (gr.next()) {
            var option = gr.sc_item_option.getRefRecord();
            if (option.item_option_new.name == variableName) {
                gr.sc_item_option.value = value;
                gr.update();
                found = true;
                break;
            }
        }
        
        if (!found) {
            var varDef = new GlideRecord('item_option_new');
            varDef.addQuery('name', variableName);
            varDef.query();
            
            if (varDef.next()) {
                var newMtom = new GlideRecord('sc_item_option_mtom');
                newMtom.initialize();
                newMtom.request_item = ritm.sys_id;
                
                var newOption = new GlideRecord('sc_item_option');
                newOption.initialize();
                newOption.item_option_new = varDef.sys_id;
                newOption.value = value;
                var optionId = newOption.insert();
                
                newMtom.sc_item_option = optionId;
                newMtom.insert();
            }
        }
    },
    
    _checkAndCloseRequest: function(requestId) {
        var openItems = new GlideAggregate('sc_req_item');
        openItems.addQuery('request', requestId);
        openItems.addQuery('state', 'NOT IN', '3,4,7');
        openItems.addAggregate('COUNT');
        openItems.query();
        
        if (openItems.next() && openItems.getAggregate('COUNT') == 0) {
            var req = new GlideRecord('sc_request');
            if (req.get(requestId)) {
                req.request_state = 'closed_complete';
                req.update();
            }
        }
    },
    
    type: 'RITMApprovalUtil'
});
```

### Appendix B: Variable Mapping

| UI Page Field | RITM Variable | REST Payload Key |
|--------------|---------------|------------------|
| Computer Name | computer_name | computer_name |
| Display Name | display_name | display_name |
| Product Version | product_version | product_version |
| Product Name | product_name | product_name |
| File Package | file_package | file_package |
| Package Name | package_name | package_name |
| Username | username | username |
| Publisher | publisher | publisher |
| Original File Name | original_file_name | original_file_name |
| File Name | file_name | file_name |
| Source Type | source_type | source_type |
| Source Name | source_name | source_name |
| File Location | file_location | file_location |
| File Size | file_size | file_size |
| Access Type | access_type | access_type |
| Access Target Type | access_target_type | access_target_type |
| Agent ID | agent_id | agent_id |
| Checksum | checksum | checksum |
| Justification | justification | justification |
| Approval Type | approved_type | approved_type |
| Approval Length | approved_length | approved_length |

### Appendix C: REST Payload Structure

```json
{
  "ritm_data": {
    "sys_id": "abc123...",
    "number": "RITM0001234",
    "short_description": "EPM Elevation Request",
    "description": "Request description",
    "state": "3",
    "state_label": "Closed Complete",
    "stage": "fulfilled",
    "priority": "3",
    "assigned_to": "John Doe",
    "assignment_group": "Service Desk",
    "opened_by": "Jane Smith",
    "opened_at": "2025-10-20 10:00:00",
    "closed_at": "2025-10-24 14:30:00",
    "closed_by": "John Approver",
    "close_notes": "Approved for 12 months by john.approver",
    "request": "REQ0001234",
    "requested_for": "Jane Smith",
    "catalog_item": "EPM Elevation Request"
  },
  "variables": {
    "computer_name": "WORKSTATION01",
    "display_name": "Application Name",
    "product_version": "1.0.0",
    "product_name": "Product Name",
    "file_package": "package.msi",
    "package_name": "Package Name",
    "username": "jsmith",
    "publisher": "Software Publisher",
    "original_file_name": "app.exe",
    "file_name": "application.exe",
    "source_type": "MSI",
    "source_name": "Internal",
    "file_location": "C:\\Program Files\\App",
    "file_size": "1024KB",
    "access_type": "Admin",
    "access_target_type": "Local",
    "agent_id": "AGENT001",
    "checksum": "abc123def456",
    "justification": "Business justification text",
    "approved_type": "month",
    "approved_length": "12"
  }
}
```

**Note**: The `approved_type` field will contain one of three values:
- `"hour"` - for hours-based approvals
- `"day"` - for days-based approvals  
- `"month"` - for months-based approvals

The `approved_length` field contains the numeric duration as a string.

### Appendix D: State Values

| State | Value | Description |
|-------|-------|-------------|
| Open | 1 | Initial state |
| Work in Progress | 2 | Being worked on |
| Closed Complete | 3 | Successfully completed |
| Closed Incomplete | 4 | Cancelled/Rejected |
| Closed Skipped | 7 | Skipped |

---

## Support and Maintenance

### Contact Information
- ServiceNow Administrator: [Your contact]
- External API Support: [API provider contact]

### Version History
| Version | Date | Changes | Author |
|---------|------|---------|--------|
| 1.0 | 2025-10-24 | Initial implementation | [Your name] |

### Known Limitations
- Button only appears on "EPM Elevation Request" catalog items
- Maximum approval duration limits:
  - Hours: 24 maximum
  - Days: 30 maximum
  - Months: 12 maximum
- REST integration is one-way (outbound only)
- No automatic retry mechanism for failed REST calls
- Approval type stored as singular value (hour/day/month) but displayed as plural

### Future Enhancements
- Add approval workflow notifications
- Implement retry logic for failed REST calls
- Add approval history tracking with duration types
- Create dashboard for approval metrics (breakdown by hours/days/months)
- Support for bulk approvals
- Custom approval duration limits per catalog item
- Email notifications with approval duration details

---

**Document End**
