# nr-showback Roadmap

This document tracks planned improvements and enhancements for the New Relic showback dashboard project.

## Upcoming Features & Improvements

### 1. Dynamic Provider Region Configuration
**Priority:** Medium  
**Status:** Planned  
**Description:** Make the New Relic provider region configurable via variables instead of hardcoded.

**Implementation Plan:**
- [ ] Add `newrelic_region` variable to `variables.tf` with validation (US/EU only)
- [ ] Update `provider.tf` to use `var.newrelic_region` instead of hardcoded "US"
- [ ] Update all `runtf-*.sh` wrapper scripts to set `TF_VAR_newrelic_region`
- [ ] Add region configuration to `terraform.tfvars.sample`
- [ ] Update documentation with region configuration examples

**Benefits:**
- Supports multi-region deployments
- Environment-specific region configuration
- Better flexibility for different New Relic accounts
- Maintains consistency across environments

**Files to modify:**
- `variables.tf` - Add region variable with validation
- `provider.tf` - Replace hardcoded region with variable
- `runtf-*.sh` scripts - Add TF_VAR_newrelic_region export
- `terraform.tfvars.sample` - Add region example
- `README.md` - Document region configuration

### 2. Regional Synthetics Script Execution
**Priority:** Medium  
**Status:** Planned  
**Description:** Configure synthetics monitoring scripts to run from geographically appropriate locations based on New Relic account region.

**Implementation Plan:**
- [ ] Add location-based synthetics configuration
- [ ] Configure Washington DC location for US region accounts
- [ ] Configure Frankfurt location for EU region accounts
- [ ] Update synthetics script deployment logic
- [ ] Add location validation and error handling

**Benefits:**
- Improved monitoring accuracy for regional services
- Reduced network latency for synthetics checks
- Compliance with data residency requirements
- Better user experience simulation from appropriate geolocations

**Technical Details:**
- US Region accounts → Washington DC synthetics location
- EU Region accounts → Frankfurt synthetics location
- Automatic location selection based on provider region variable
- Fallback logic for unsupported regions

---

## Recently Completed

### ✅ Synthetics Cost Tracking Feature
**Completed:** October 2025  
**Description:** Added comprehensive synthetics monitoring cost tracking with minimum commitment pricing support.

**Features delivered:**
- Synthetics cost variables with validation
- Enhanced dashboard widgets with synthetics data
- Department-based synthetics cost allocation
- Monthly consumption tracking with synthetics
- Backward compatibility maintained
- Dual-query architecture preserved

---

## Future Considerations

### Additional Metrics Support
- Browser monitoring costs
- Mobile monitoring costs  
- Infrastructure monitoring costs
- Custom events pricing

### Dashboard Enhancements
- Cost forecasting widgets
- Budget alerts integration
- Department-level cost controls
- Export functionality for finance teams

### Infrastructure Improvements
- Multi-environment support (dev/staging/prod)
- Automated testing for dashboard widgets
- CI/CD pipeline for deployments
- Terraform state management improvements

---

*Last updated: October 9, 2025*