[![New Relic Experimental header](https://github.com/newrelic/opensource-website/raw/master/src/images/categories/Experimental.png)](http#### **How Minimum Commitment Pricing Works**
- **Committed Cost**: `minimum_amount × committed_rate`
- **Additional Cost**: `MAX(0, actual_usage - minimum_amount) × additional_rate`  
- **Total Cost**: `committed_cost + additional_cost`

**Example**: With 12 core users (10 minimum at $45, 2 additional at $49):
- Committed: 10 × $45 = $450
- Additional: 2 × $49 = $98  
- Total: $548ource.newrelic.com/oss-category/#new-relic-experimental)
# nr-showback

This repository provides an automated way to report New Relic ingest consumption and user costs, by business department, using aggregated account-based cost allocation. **Now with support for minimum commitment pricing** for enterprise contract billing models.

![Data flow diagram](screenshots/nr-showback-data-flow-diagram.png)

A single [terraform.tfvars](terraform.tfvars) file contains the definition of departments within a business, a customer's prices for ingest and user consumption, minimum commitment amounts, and the ability to ignore certain groups. Applying the terraform via a wrapper script creates a synthetics script, secure credentials containing API keys, and an associated dashboard. Once every 24 hours, the synthetics script queries the New Relic GraphQL API for a customer's organization and user management data structures. Based upon a model of hierarchical account-based cost allocation, showback data is posted into NRDB as metrics, and user data as custom events. To view the showback data, customers access a dashboard that is built, and kept in sync with the departmental definitions, by terraform configuration.

![Example dashboard](screenshots/nr-showback-dashboard.png)

## ✨ New: Minimum Commitment Pricing

This solution now supports **minimum commitment pricing models** commonly used in enterprise New Relic contracts:

- **Committed Rates**: Different pricing for committed minimums vs. additional usage
- **Flexible Minimums**: Set minimum committed users and data ingest amounts
- **Automatic Calculations**: Dashboard automatically calculates committed costs + additional costs
- **Backward Compatible**: Existing configurations continue to work unchanged

### Quick Example
```hcl
showback_price = {
  # Minimum commitments
  min_core_users = 4      # 4 core users minimum
  min_full_users = 9      # 9 full users minimum  
  min_gb_ingest = 2500    # 2500GB ingest minimum
  
  # Committed rates (lower cost for minimums)
  core_user_committed_usd = 49
  full_user_committed_usd = 315
  gb_ingest_committed_usd = 0.35
  
  # Additional rates (standard cost for overage)
  core_user_additional_usd = 49
  full_user_additional_usd = 310
  gb_ingest_additional_usd = 0.35
}
```

**Result**: Pay committed rates for minimums, then standard rates for any usage above minimums.

## Will it work for *us*?
### Good fit
Assuming that a customer has created a hierarchical account structure, where each department has one or more accounts, it is possible to aggregate costs associated with each account at a departmental level. Let’s call these hierarchical account structures. Customers with hierarchical account structures are a good fit for the automation provided in this repo.

<img src="screenshots/good-fit.png" alt="Good fit" width="300"/>

### Poor fit
Alternatively, a customer might have a simple account structure, where all departments share a single account, or common Prod/QA/Dev accounts for example. Let’s call these non-hierarchical account structures. These are a poor fit for the approach described here. 

<img src="screenshots/poor-fit.png" alt="Poor fit" width="220"/>

## User apportioning method
This showback solution uses a user apportioning method. It works as follows:
- If a user is in a single account, their user is allocated to that account
- If a user is in more than one account, their user is apportioned to each account equally. For example, a user in two accounts will have ½ a user allocated to each account; and a user in three accounts will have ⅓ of a user apportioned to each account.

Specific groups can be ignored if, say, all users are members of a group with read-access to all accounts.

## Installation
Make sure terraform is installed. We recommend [tfenv](https://github.com/tfutils/tfenv) for managing your terraform binaries.

Update the [runtf.sh.sample](runtf.sh.sample) wrapper file with your credentials and account details and rename it `runtf.sh`.Alternatively, if you are running from Windows: update the [runtf.bat.sample](runtf.bat.sample) wrapper file with your credentials and account details and rename it `runtf.bat`.  **Important do not commit this new file to git!** (It should be ignored in `.gitignore` already.) 

The wrapper file contains configuration of three API keys:
1.  `NEW_RELIC_API_KEY`: a User API key to create terraform resources
2.  `TF_VAR_showback_query_user_api_key`: a User API key to query user management configuration in GraphQL - stored as a secure credential in Synthetics under the name `SHOWBACK_QUERY_USER_API_KEY`
3.  `TF_VAR_showback_insert_license_api_key`: an Ingest API key for posting showback and user data in NRDB - stored as a secure credential in Synthetics under the name `SHOWBACK_INSERT_LICENSE_API_KEY`

The user associated with the `TF_VAR_showback_query_user_api_key` variable must have a user type of Full or Core, and be a member of a group with Organization and Authentication Domain [Administration settings](https://docs.newrelic.com/docs/accounts/accounts-billing/new-relic-one-user-management/user-management-concepts#admin-settings) enabled.

The account IDs used for the terraform resources, billing account, and reporting account may be different, but are all likely to be the billing account.

The wrapper file also allows the configuration of the following:
1.  `TF_VAR_monitor_name`: the name of the showback reporting script on the reporting account
2.  `TF_VAR_dashboard_name`: the name of the showback dashboard on the reporting account
3.  `TF_VAR_event_name_prefix`: the prefix used in nr-showback metric names, defaults to "Showback", resulting in events of the form "Showback_UniqueUsers" for example. It is recommended that this value is modified during testing, e.g. to TestShowback, and reverted for production use
4.  `TF_VAR_metric_name_prefix`: the prefix used in nr-showback metric names, defaults to "showback", resulting in metrics of the form "showback.department.fulluser.count" for example. It is recommended that this value is modified during testing, e.g. to test.showback, and reverted for production use

Note: You may want to update the version numbers in [provider.tf](provider.tf) and [modules/monitor/provider.tf](modules/monitor/provider.tf) to the latest versions of Terraform and the New Relic provider. You will need to update both provider.tf files if you are using the EU region.

## Showback configuration
The showback configuration is entirely within the terraform.tfvars file. Copy [terraform.tfvars.sample](terraform.tfvars.sample), which is populated with an example config to a file named `terraform.tfvars`. Modify the configuration for your account. The configuration contains:
- `showback_price`: the costs for:
  - full users (`full_user_usd`)
  - core users (`core_user_usd`)
  - billable ingest per GB (`gb_ingest_usd`)
- `showback_ignore.groups`: whether specific user group membership should be ignored. Some customers grant read-only access to all accounts, which breaks the script’s showback user apportioning
- `showback_ignore.newrelic`: whether New Relic employees should be ignored in the showback charge, set to `true`, but can be changed
- `showback_config`: for each department, the `department_name`, an optional `tier` value (for grouping departments into higher level reporting units), and accounts either as a list (`accounts_in`) or as a list of one or more regular expressions (`accounts_regex`)

The expectation with the tier value is that all accounts are separately mapped to one or more reporting units. Any additional tiers will be displayed on a separate page on the dashboard with the page title set to the tier name, e.g. 'Reporting Unit'.

## 💰 Pricing Models

### 1. **Simple Pricing** (Original)
Basic per-unit pricing for straightforward billing:
```hcl
showback_price = {
  core_user_usd = 49        # $49 per core user
  full_user_usd = 99        # $99 per full user  
  gb_ingest_usd = 0.30      # $0.30 per GB ingest
}
```

### 2. **Minimum Commitment Pricing** (New)
Enterprise contract pricing with committed minimums and overage rates:
#### 2. **Minimum Commitment Pricing** (New)
Enterprise contract pricing with committed minimums and additional rates:
```hcl
showback_price = {
  # Legacy rates (used as fallback)
  core_user_usd = 49
  full_user_usd = 99  
  gb_ingest_usd = 0.30
  
  # Minimum commitments
  min_core_users = 10       # Minimum 10 core users
  min_full_users = 5        # Minimum 5 full users
  min_gb_ingest = 1000      # Minimum 1000GB ingest
  
  # Committed rates (discounted pricing for minimums)
  core_user_committed_usd = 45
  full_user_committed_usd = 90
  gb_ingest_committed_usd = 0.25
  
  # Additional rates (standard pricing for additional usage)
  core_user_additional_usd = 49
  full_user_additional_usd = 99
  gb_ingest_additional_usd = 0.30
  
  # Optional: prorate minimums for partial months
  prorate_minimums = false
}
```

### **How Minimum Commitment Pricing Works**
- **Committed Cost**: `minimum_amount × committed_rate`
- **Overage Cost**: `MAX(0, actual_usage - minimum_amount) × additional_rate`  
- **Total Cost**: `committed_cost + overage_cost`

**Example**: With 12 core users (10 minimum at $45, 2 additional at $49):
- Committed: 10 × $45 = $450
- Overage: 2 × $49 = $98  
- Total: $548

### **Key Features**
- ✅ **Backward Compatible**: Existing configurations continue to work unchanged
- ✅ **Flexible Pricing**: Different rates for committed vs. additional usage
- ✅ **Enterprise Ready**: Supports complex contract pricing models
- ✅ **Automatic Calculations**: Dashboard handles all commitment logic
- ✅ **Validation**: Terraform ensures configuration correctness

See [terraform.tfvars.minimum-commit-example](terraform.tfvars.minimum-commit-example) for a complete working example.

## Initialization
Use the `runtf.sh` helper script wherever you would normally run `terraform`. It simply wraps the terraform with some environment variables that make it easier to switch between projects. (You don't have to do it this way, you could just set the env vars and run terraform normally.)

First initialise terraform:
```
./runtf.sh init
```

Now apply the changes:
```
./runtf.sh apply
```

## State storage
This example does not include remote state storage. State will be stored locally in `terraform.tfstate`.

## Showback NRDB data types
The synthetics script, default name NR Showback reporting script, posts four types of data back to NRDB, they are:
1. `showback.department.*` and aggregated `showback.organization.*` metrics, containing breakdowns of the number of users by type at the department and organization levels.
2. `showback.account.*` metrics, containing breakdowns of the number of users by type at an account level.
3. `Showback_UniqueUsers` custom events, containing an event per unique user with their last access time, user type and a set of the departments to which they have been allocated.
4. `Showback_AccountUsers` custom events, containing an event per user, per role, per account.

## Dashboard reporting
The dashboard, default name `NR Showback reporting`, contains three pages with **enhanced cost calculations** that automatically support both simple and minimum commitment pricing:

1. `Department Showback` (shown above)
- **Cost breakdown** by department with automatic commitment/additional calculations
- **Pie charts** showing cost distribution across departments
- **Detailed table** with individual cost components (Core, Full, Ingest costs)
- **User distribution** widgets showing breakdown by type per department  
- **Monthly trending** table showing historical consumption with commitment-based billing
2. `Account Users (Summary)`
- **Tabular breakdown** of users per account with cost implications
- **User type distribution** charts showing % breakdown and trends over time
- **Unique users table** listing each user by email address with department allocations
3. `Account Users (All Accounts)`  
- **Account overview** with user count summaries
- **Comprehensive user list** showing every user, role, and account access

### **Enhanced Cost Features**
All cost calculations automatically adapt based on your pricing configuration:
- **Simple pricing**: Direct multiplication of usage × rates
- **Commitment pricing**: Committed minimums + additional calculations
- **Mixed scenarios**: Handles partial commitments (e.g., only Core user minimums)
- **Real-time updates**: Dashboard reflects current pricing model without modification

## 🔧 Technical Implementation

### **Files Modified for Minimum Commitment Support**
1. **`variables.tf`** - Enhanced pricing variable definitions with validation
2. **`terraform.tfvars.sample`** - Updated sample configuration with examples
3. **`dashboard.tf`** - Extended template variable passing for new pricing parameters
4. **`dashboards/dashboard.json.tftpl`** - Advanced NRQL cost calculations with conditional logic
5. **`terraform.tfvars.minimum-commit-example`** - Complete working example (new)

### **NRQL Implementation Details**
The dashboard uses sophisticated NRQL queries with conditional logic:
```sql
-- Example: Core user cost calculation
(${tf_min_core_users} * ${tf_core_user_committed_usd} + 
 if(max(core_user_count) > ${tf_min_core_users}, 
    (max(core_user_count) - ${tf_min_core_users}) * ${tf_core_user_additional_usd}, 0))
```

This automatically:
- ✅ Calculates committed cost for minimums
- ✅ Adds additional cost for usage above minimums  
- ✅ Handles edge cases (usage below minimums)
- ✅ Maintains backward compatibility with simple pricing

### **Validation & Safety**
- **Terraform validation** ensures minimum commitments are non-negative
- **Configuration validation** requires both committed and additional rates when minimums are set
- **NRQL error handling** prevents negative additional costs and division by zero
- **Backward compatibility** preserved for existing configurations

# Support

New Relic has open-sourced this project. This project is provided AS-IS WITHOUT WARRANTY OR DEDICATED SUPPORT. Issues and contributions should be reported to the project here on GitHub.

We encourage you to bring your experiences and questions to the [Explorers Hub](https://discuss.newrelic.com) where our community members collaborate on solutions and new ideas.

## Issues / enhancement requests

Issues and enhancement requests can be submitted in the [Issues tab of this repository](../../issues). Please search for and review the existing open issues before submitting a new issue.

# Contributing

Contributions are encouraged! If you submit an enhancement request, we'll invite you to contribute the change yourself. Please review our [Contributors Guide](CONTRIBUTING.md).

Keep in mind that when you submit your pull request, you'll need to sign the CLA via the click-through using CLA-Assistant.

# Open source license
This project is distributed under the [Apache 2 license](LICENSE).
