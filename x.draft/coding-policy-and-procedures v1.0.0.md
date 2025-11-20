# Coding Policy and Procedures for One Way Path Communities

## Table of Contents
1. Purpose & Scope
2. Development Process & "Don't Reinvent the Wheel"
   - 2.1 Tool Selection Order
   - 2.2 When to Move to Custom Code
3. Coding Plan & Repository Setup
   - 3.1 Coding Plan Requirements
   - 3.2 Module-Level Plans
4. Code Organization, Modularity & Naming
   - 4.1 Modular by Function
   - 4.2 Naming Conventions
   - 4.3 Stable Releases & Versioning
5. API Integration & Express Server Pattern
   - 5.1 Every Completed Module Has an API Surface
   - 5.2 Route Registration: Hybrid Helper Pattern
   - 5.3 Environment Variables & .env Loading
6. Documentation Standards
   - 6.1 README Files
   - 6.2 Function-Level Documentation (JSDoc)
   - 6.3 Embedded Self-Tests
7. Code Inside Hosted Apps (Airtable, Asana, Office)
8. ChatGPT Procedural Checklist
9. Appendix: Example Snippets

---

# 1. Purpose & Scope
This document defines the standards, workflow practices, naming conventions, coding structure, documentation requirements, and API integration rules applicable to all software written for OWP projects. It applies to Node/Express servers, JavaScript modules, Python modules, and scripts inside hosted apps.

# 2. Development Process & "Don't Reinvent the Wheel"
Before writing custom code, always choose the simplest viable tool.

## 2.1 Tool Selection Order
1. Existing app features  
2. Automations inside apps  
3. Zapier zaps  
4. Custom API coding  

## 2.2 When to Move to Custom Code
Use custom code only when simpler tools cannot handle complexity or reliability needs.

# 3. Coding Plan & Repository Setup

## 3.1 Coding Plan Requirements
Every repo must begin with a CODING_PLAN.md containing:
- Problem statement  
- Planned modules  
- API surfaces  
- Environment variables  
- Testing strategy  
- Release plan  

## 3.2 Module-Level Plans
Each module needs a short definition of purpose, exports, and API exposure.

# 4. Code Organization, Modularity & Naming

## 4.1 Modular by Function
One responsibility per module. Export pure functions where possible.

## 4.2 Naming Conventions
- Database: lower_snake_case  
- JS code files: camelCase OR kebab-case for routes  
- Folders: *JS for JS, *PT for Python  
- URLs: kebab-case  

## 4.3 Stable Releases & Versioning
Each repo/module must have at least one stable release (e.g., v1.0.0).

# 5. API Integration & Express Server Pattern

## 5.1 Every Completed Module Has an API Surface
Each module intended for reuse must expose an API route for testing and automation.

## 5.2 Route Registration: Hybrid Helper Pattern
routes/registerRoutes.mjs:
```js
import acmeRouter from './acme.mjs';
import devBranchDeployRouter from './github-webhooks/serverJS-dev-branch-deploy.mjs';
import scorecardUploadRouter from './golfScorePostJS/scorecard-upload.mjs';
import bvCampaignRouter from './bv-campaign.mjs';

export function registerRoutes(app) {
  app.use(acmeRouter);
  app.use(devBranchDeployRouter);
  app.use(scorecardUploadRouter);
  app.use(bvCampaignRouter);
}
```

server.mjs:
```js
import express from 'express';
import { registerRoutes } from './routes/registerRoutes.mjs';

const app = express();
registerRoutes(app);
export default app;
```

## 5.3 Environment Variables & .env Loading
Each module loads its own .env so server.mjs never needs modification.

# 6. Documentation Standards

## 6.1 README Files
Every repo/module requires a README with a Table of Contents and API/usage instructions.

## 6.2 Function-Level Documentation (JSDoc)
Each exported function requires:
```js
/**
 * Summary description
 * @param {...}
 * @returns {...}
 * Example usage:
 */
```

## 6.3 Embedded Self-Tests
Each module must include a self-test using:
```js
if (import.meta.url === `file://${process.argv[1]}`) { ... }
```

# 7. Code Inside Hosted Apps (Airtable, Asana, Office)
Scripts must follow modular structure and documentation style similar to standalone modules.

# 8. ChatGPT Procedural Checklist
ChatGPT must:
- Check simpler options first  
- Produce/update coding plan  
- Follow naming conventions  
- Ensure modularity  
- Add/modify API route + registration  
- Keep env loading in modules  
- Provide README  
- Suggest stable release tagging  
- Validate compliance with this policy  

# 9. Appendix: Example Snippets
## Example JSDoc + Self-Test
```js
export function parseRoundInfo(text) { return {}; }

if (import.meta.url === `file://${process.argv[1]}`) {
  console.log(parseRoundInfo("demo text"));
}
```
