## Dynamics 365 Project Operations - Employee financial dimensions (example solution)

*This is an example customisation for [Dynamics 365 Project Operations](https://dynamics.microsoft.com/en-us/project-operations/overview/). The solution is provided 'as-is' with no warranty and is not supported by Microsoft. It can be used as a guide to build out a custom solution for a customer implementation, to be managed/maintained by the customer or by the implementing partner. Check out the CE and FinOps branches in this repo to browse the code/artefacts.*

**NOTE** - THIS IS WORK IN PROGRESS (27.11.2020)

Here we will talk about a solution for a feature gap that many customers will face in Dynamics 365 Project Operations [integrated deployment mode](https://docs.microsoft.com/en-us/dynamics365/project-operations/environment/project-operations-integrated-deployment-overview) (as per November 2020 release,) but this is also a useful example of how to implement a customisation that spans the D365 Customer Engagement and Finance & Operations platforms, using [DualWrite integration](https://docs.microsoft.com/en-us/dynamics365/fin-ops-core/dev-itpro/data-entities/dual-write/dual-write-overview). Let's get into it!

### Description of gap

As a consultant in a professional services business, I am logging the time that I spend working on specific tasks on specific projects day by day. I do this in the Project Operations model drivel app on the CE platform.

As a project accountant I receive transactions, including the time entries mentioned above, as a journal in the Project module of F&O. The journal is created for me automatically by the out of the box integration between the two systems and I must ensure that the journal has the correct accounting distributions before posting it.

As a financial controller of the organisation, I want project actuals to be posted to the general ledger stamped with financial dimensions such as department or cost centre, which are associated with the worker who recorded the time on a particular task. These dimension values may be different to the default dimensions associated with the project itself and would be configured against the worker's employment record in F&O:

![Screenshot of employee default financial dimensions in Finance & Operations](https://github.com/finopsfuntimes/ProjectOpsEmpFinDims/raw/main/ScreenShots/FinOpsEmployeeFinDims.JPG)

In the current release of D365 Project Operations (November 2020,) there is no direct link between worker records in the Finance & Operations system (where the default financial dimensions are set,) and the 'bookable resources' in the Project Operations Customer Engagement based app who record their time on a project. This means that **the default financial dimensions set on the automatically created journal lines do not include the employee's default financial dimension values**.

*This gap should be addressed in a future release of D365 Project Operations, building on improved DualWrite integration with core HR but for the short to medium term, the solution will need to be extended to meet the above requirement.*

### Proposed solution

Firstly, we recognise that the objective of this extension is only to facilitate the defaulting of financial dimensions from the worker/employee record down to the GL transactions that will be posted in F&O. The project accountant always has the ability to set/override the dimension values prior to posting. We also recognise that we aren't trying to build broader integration of the HR module with CE.

To achieve the goal, we require that when project actuals records are passed from CE to F&O via the DualWrite integration, the records should somehow identify who did the work. This would then allow some custom business logic in F&O to lookup the worker and copy their default financial dimensions onto the journal line, which is used to post those actuals to the general ledger.

Time entries are recorded in the CE based app by a bookable resource. This is usually a team member who logs into the system and records their own time. Our first stated assumption is that the bookable resources are indeed users of the system and therefore that the bookable resource records are of type 'user'.

![Screenshot of bookable resource details form in Project Operations CE app](https://github.com/finopsfuntimes/ProjectOpsEmpFinDims/raw/main/ScreenShots/CEBookableResourceDetail.JPG)

We need to have some way of linking a bookable resource in the CE based app with the worker record in F&O. For this purpose, we can take advantage of the fact that bookable resource records of type 'user' in CE already have a field 'Primary Email'. The field is populated automatically (copied from the 'Primary Email Address' field of the AAD account of the user,) when the bookable resource is created as a 'user' type.

Our assumptions so far:

 - Bookable resources are of type 'user'
 - The users will have a 'primary email address' populated on their Active Directory account
 - The same email address will be used to setup the user in the F&O system

If these assumptions are not true in your case, then an alternative ID would be needed to link the bookable resource and the employee. A new custom field on the bookable resource record could store the F&O worker ID, for example. The rest of the solution can then be easily adapted to make use of this ID instead.

When the approved time entries are created in the CE app, they are associated with the bookable resource who completed those hours. Currently, this reference is only being passed over to the F&O system as part of a description field that would be printed on the invoice. The proposed solution is to incorporate an additional field in both CE and F&O on the actuals tables to hold the email address of the worker and include it in the mapping for the integration between the systems.

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

 - *Links to be provided*
 - Create a publisher
 - Create an unmanaged solution

##### 2. Prepare the F&O project
 - *Links to be provided*
 - Create a new model with references
 - Create a project 

#### Part 1 - Add the email field to to the actuals table in CE and populate automatically
In the PowerApps maker portal, add the table 'Actual' `msdyn_actual` to your unmanaged solution and do not include any components:
![Screenshot of PowerApps maker portal, added Actual table to unmanaged solution](https://github.com/finopsfuntimes/ProjectOpsEmpFinDims/raw/main/ScreenShots/MakeAddActualTable.JPG)

Open the table details and add a new column to hold the email address of the bookable resource. The data type should be text, and this will be a calculated field so click the 'Add' button to create the calculation. The calculation will lookup the email address of the bookable resource using the action formula `msdyn_bookableresource.msdyn_primaryemail` as per the screenshot:
![Screenshots of adding calculated column to Actuals table in PowerApps maker portal](https://github.com/finopsfuntimes/ProjectOpsEmpFinDims/raw/main/ScreenShots/MakeAddEmailField.JPG) 

Save the changes and publish the customisation.
#### Part 2 - Extend the F&O data entity with an email field
Table`ProjCDSActualsImport`
Table`ProjActualsImportStaging`
Entity `ProjActualsImportEntity`
#### Part 3 - Incorporate the new field into the DualWrite map
#### Part 4 - Extend F&O business logic to lookup default financial dimensions
Class `ProjCDSActualsImportIntegration`
