# Resources — Israeli Tax Refund Flow

Links to learn about the refund process, the tax authority portal, and the underlying regulations.

---

## Official Government Portals

| Resource                                   | URL                                             | Notes                                            |
| ------------------------------------------ | ----------------------------------------------- | ------------------------------------------------ |
| Israeli Tax Authority (רשות המסים)         | https://www.misim.gov.il                        | Main portal for all tax filings                  |
| Personal tax refund submission             | https://www.gov.il/he/service/itc135            | Direct entry point for refund requests (החזר מס) |
| Income tax portal (mas hakhnasa)           | https://secapp.taxes.gov.il/SrRishum/           | Authenticated zone for income tax                |
| National Insurance Institute (ביטוח לאומי) | https://www.btl.gov.il                          | Relevant for deductions and credits              |
| Gov.il — tax refund guide                  | https://www.gov.il/he/service/income_tax_refund | Step-by-step citizen guide                       |

---

## Understanding the Refund Flow

| Resource                                          | URL                                                            | Notes                                                |
| ------------------------------------------------- | -------------------------------------------------------------- | ---------------------------------------------------- |
| Who is entitled to a tax refund? (gov.il)         | https://www.gov.il/he/departments/guides/guide_tax_refund      | Eligibility criteria                                 |
| Submitting a refund request — Tax Authority guide | https://taxes.gov.il/incometax/pages/individualstaxrefund.aspx | Official walkthrough                                 |
| Form 106 (טופס 106) explained                     | https://www.gov.il/he/service/itc135                           | Annual employer wage certificate required for filing |
| Annual tax report for employees (דו"ח שנתי)       | https://www.gov.il/he/service/annual_tax_return_for_employee   | When and how employees file                          |
| Rulebase for exempt income and deductions         | https://taxes.gov.il/incometax/pages/taxrates.aspx             | Tax brackets, credits, and exemptions                |

---

## Authentication & Login Flow

| Resource                                | URL                                                     | Notes                                        |
| --------------------------------------- | ------------------------------------------------------- | -------------------------------------------- |
| GovID / digital ID login (Gov.il login) | https://www.gov.il/he/departments/department/digital-id | Understand the OTP-based login used by misim |
| Misim login page                        | https://secapp.taxes.gov.il/SrRishum/                   | Entry point Claude navigates to              |

---

## Technical / Automation Reference

| Resource                   | URL                                                | Notes                                     |
| -------------------------- | -------------------------------------------------- | ----------------------------------------- |
| Playwright MCP (Microsoft) | https://github.com/microsoft/playwright-mcp        | MCP server used for browser control       |
| Playwright docs            | https://playwright.dev/docs/intro                  | Browser automation framework              |
| Claude Code MCP guide      | https://docs.anthropic.com/en/docs/claude-code/mcp | How to connect MCP servers in Claude Code |
| Anthropic Claude Code docs | https://docs.anthropic.com/en/docs/claude-code     | Claude Code overview                      |

---

## Community & Background Reading

| Resource                                         | URL                                  | Notes                                       |
| ------------------------------------------------ | ------------------------------------ | ------------------------------------------- |
| taxes-refund.co.il — Israeli tax refund guide (Hebrew) | https://taxes-refund.co.il/                                                                                       | Popular consumer guide                      |
| Mako Money — who should request a refund               | https://www.mako.co.il/finances-money                                                                             | General audience articles on annual refunds |
| taxes-refund.co.il refund simulator                    | https://taxes-refund.co.il/%D7%A1%D7%99%D7%9E%D7%95%D7%9C%D7%98%D7%95%D7%A8-%D7%94%D7%97%D7%96%D7%A8-%D7%9E%D7%A1-%D7%9E%D7%97%D7%A9%D7%91%D7%95%D7%9F-%D7%94%D7%97%D7%96%D7%A8-%D7%9E%D7%A1/ | Estimate your expected refund               |

---

## Key Concepts Glossary

| Hebrew Term  | English           | Relevance                                                   |
| ------------ | ----------------- | ----------------------------------------------------------- |
| החזר מס      | Tax refund        | The core subject of this project                            |
| טופס 106     | Form 106          | Annual wage certificate from employer — required for filing |
| נקודות זיכוי | Tax credit points | Used to reduce tax liability; eligibility varies            |
| מספר זהות    | ID number         | Primary identifier for authentication                       |
| דו"ח שנתי    | Annual tax report | The form submitted to claim the refund                      |
| מס הכנסה     | Income tax        | The tax type being refunded                                 |
| רשות המסים   | Tax Authority     | The government body managing the process                    |
| אתר מיסים    | Misim portal      | The web portal this project automates                       |
