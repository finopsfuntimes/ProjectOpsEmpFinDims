## Dynamics 365 Project Operations - Employee financial dimensions (example solution)

*This is an example customisation for [Dynamics 365 Project Operations](https://dynamics.microsoft.com/en-us/project-operations/overview/). The solution is provided 'as-is' with no warranty and is not supported by Microsoft. It can be used as a guide to build out a custom solution for a customer implementation, to be managed/maintained by the customer or by the implementing partner. Check out the CE and FinOps branches in this repo to browse the code/artefacts.*

**NOTE** - THIS IS WORK IN PROGRESS (26.11.2020)

Here we will talk about a solution for a feature gap that many customers will face in Dynamics 365 Project Operations [integrated deployment mode](https://docs.microsoft.com/en-us/dynamics365/project-operations/environment/project-operations-integrated-deployment-overview) (as per November 2020 release,) but this is also a useful example of how to implement a customisation that spans the D365 Customer Engagement and Finance & Operations platforms, using [DualWrite integration](https://docs.microsoft.com/en-us/dynamics365/fin-ops-core/dev-itpro/data-entities/dual-write/dual-write-overview). Let's get into it!

### Description of gap

It's a common customer requirement to want to post project actuals to the general ledger with financial dimensions such as department or cost centre, which are associated with the worker who booked the time on a particular task. These dimension values may be different to the default dimensions associated with the project itself and would be configured against the worker's employment record in F&O:

![Screenshot of employee default financial dimensions in Finance & Operations](https://github.com/finopsfuntimes/ProjectOpsEmpFinDims/raw/main/ScreenShots/FinOpsEmployeeFinDims.JPG)

In the current release of D365 Project Operations (November 2020,) there is no direct link between worker records in the Finance & Operations system and the bookable resources in the Project Operations Customer Engagement based app who record their time on a project. This simply means that the default financial dimensions on the transaction are only coming from the project and the customer records.

*This gap should be addressed in a future release of Project Operations, building on improved DualWrite integration with core HR but for the short to medium term, the solution will need to be extended to meet the above requirement.*

### Proposed solution

Firstly, we recognise that the objective of this extension is only to facilitate the defaulting of financial dimensions from the worker/employee record down to the GL transactions that will be posted in F&O. The project accountant always has the ability to set/override the dimension values prior to posting. We also recognise that we aren't trying to build broader integration of the HR module with CE.

To achieve the goal, we require that when project actuals records are passed from CE to F&O via the DualWrite integration, the records should somehow identify who did the work, which then allows our custom code in F&O to lookup the worker and copy their default financial dimensions onto the journal line, which is used to post those actuals to the general ledger.

Time entries are recorded in the CE based app against a bookable resource, this is usually a team member who logs into the system and records their own time. Our first stated assumption is that the bookable resources are indeed users of the system and therefore that the bookable resource records are of type 'user'.

![Screenshot of bookable resource details form in Project Operations CE app](https://github.com/finopsfuntimes/ProjectOpsEmpFinDims/raw/main/ScreenShots/CEBookableResourceDetail.JPG)

We need to have some way of linking a bookable resource in the CE based app with the worker record in F&O. For this purpose, we can take advantage of the fact that bookable resource records in CE already have a field 'Primary Email', which is populated from the AAD account of the user, when the bookable resource is created as a 'user' type. We can leverage this if we assume that the primary email field has been populated on the AAD account and that the same user will be setup in F&O using that same primary email address. This gives us a unique identifier to correlate bookable resource and the worker record.

*Diagram/screenshot here*

When the approved time entries are created in the CE app, they are associated with the bookable resource who completed those hours. Currently, this reference is only being passed over to the F&O system as part of a description field that would be printed on the invoice. The proposed solution is to incorporate an additional field in both CE and F&O on the transaction to hold the email address of the worker and include it in the mapping for the integration between the systems.

In standard, when the dimension defaults are applied to the transactions created in the F&O system after receiving the actuals from the CE app, they are merged from default dimension setup on the customer and project records. In the proposed solution, an extension will be written to make use of the email address to lookup and merge their default dimensions with those from the customer, project and worker.

### Design

We can break the solution down into some key components:

1.  In CE - add an email field to the actuals table and populate it automatically from the bookable resource
2.  In F&O - extend the data entity used for integration to include the email field
3.  Incorporate the new email field in the DualWrite mapping
4.  In F&O - extend the business logic to lookup the worker and add the default dimensions

*More detail to go here*

### Implementation
#### Pre-requisites
##### 1. Prepare the CE solution
##### 1. Prepare the F&O project

#### Part 1 - Add the email field to to the actuals table in CE and populate automatically
#### Part 2 - Extend the F&O data entity with an email field
Table`ProjCDSActualsImport`
Table`ProjActualsImportStaging`
Entity `ProjActualsImportEntity`
#### Part 3 - Incorporate the new field into the DualWrite map
#### Part 4 - Extend F&O business logic to lookup default financial dimensions
Class `ProjCDSActualsImportIntegration`
