## Dynamics 365 Project Operations - Employee financial dimensions (example solution)

*This is an example customisation for [Dynamics 365 Project Operations](https://dynamics.microsoft.com/en-us/project-operations/overview/). The solution is provided 'as-is' with no warranty and is not supported by Microsoft. It can be used as a guide to build out a custom solution for a customer implementation, to be managed/maintained by the customer or by the implementing partner.*

*Browse the CE_Solution and FinOps_Solution folders in this repo for source code, follow the walkthrough below to build step by step or download the latest release to get the Power Apps solution file and the F&O axpp solution file, which can be imported into your own dev environments for testing out.* 

**NOTE** - THIS IS WORK IN PROGRESS (28.11.2020)

## Walkthrough

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
It is expected that the customisation is being implemented in a sandbox CE environment connected via DualWrite to an F&O development environment and that the D365 Project Operations app has been [deployed in integrated mode](https://docs.microsoft.com/en-us/dynamics365/project-operations/environment/resource-provision-new-environment) with suitable data for testing. Validate that the approved hours transactions are flowing between the systems as expected before making any customisations. Best practices should be followed for customising each system and although the Power Platform changes are 'no-code/low-code' and therefore easily implemented by an experienced power user, the Finance & Operations changes are best performed by an [experienced developer](https://docs.microsoft.com/en-us/learn/certifications/exams/mb-500).
##### a. Prepare the CE solution
 - [Create a publisher](https://docs.microsoft.com/en-us/powerapps/maker/common-data-service/change-solution-publisher-prefix)
 - [Create an unmanaged solution](https://docs.microsoft.com/en-us/powerapps/maker/common-data-service/create-solution)

##### b. Prepare the F&O project
- [Connect to the development environment via RDP](https://docs.microsoft.com/en-us/dynamics365/fin-ops-core/dev-itpro/dev-tools/access-instances#accessing-the-cloud-instance-through-remote-desktop)
 - [Create a new model](https://docs.microsoft.com/en-us/dynamics365/fin-ops-core/dev-itpro/dev-tools/models#creating-a-new-model)
 - [Create a new project](https://docs.microsoft.com/en-us/dynamics365/fin-ops-core/dev-itpro/dev-tools/projects#create-a-new-project)

#### Part 1 - Add the email field to to the actuals table in Power Platform and populate automatically
In the [PowerApps maker portal](https://make.powerapps.com), navigate to the correct environment then add the table 'Actual' `msdyn_actual` to your unmanaged solution and do not include any components:
![Screenshot of PowerApps maker portal, added Actual table to unmanaged solution](https://github.com/finopsfuntimes/ProjectOpsEmpFinDims/raw/main/ScreenShots/MakeAddActualTable.JPG)

Open the table details and add a new column to hold the email address of the bookable resource. The data type should be text, and this will be a calculated field so click the 'Add' button to create the calculation. The calculation will lookup the email address of the bookable resource using the action formula `msdyn_bookableresource.msdyn_primaryemail` as per the screenshot:
![Screenshots of adding calculated column to Actuals table in PowerApps maker portal](https://github.com/finopsfuntimes/ProjectOpsEmpFinDims/raw/main/ScreenShots/MakeAddEmailField.JPG) 

Save the changes and publish the customisation.  

#### Part 2 - Extend the F&O data entity with an email field
The table that we need to extend with our new field in Finance & Operations is `ProjCDSActualsImport` but since we are going to need this field exposed for integration purposes, a data entity is also involved. The data entity exposes tables from the system via OData endpoints and data management framework and so we include the new custom field in the data entity and it's associated staging table, which are  `ProjActualsImportEntity` and `ProjActualsImportStaging`
respectively.

In Visual Studio on the development VM, search for the tables and data entity mentioned above one by one. As you find them, right click and select 'create extension in current project'. You should end up with a project as shown in the screenshot below:

Double click on the `ProjCDSActualsImport` table to open the designer. Search for the extended data type BaseEmail and drag it from the results window, dropping it on the 'fields' node in the designer window as shown in the screenshot below. You can then use the properties window to change the name of the new field to something appropriate.


Next we need to add the same field to the data entity staging table, which would be used by any data management import or export projects. Double click the `ProjActualsImportStaging` table from your project to open the designer and repeat the steps above to drag the BaseEmail EDT into the fields node and give it the same name as above.

At this point, we should build the solution to ensure that the new field is available for the next step.

The data entity now needs to have the new field added, so double click `ProjActualsImportEntity` in your project to open the designer. Expand the data sources node and find the new field that was added above. You can drag the field from the datasource up to the 'fields' node at the top.

Finally, we need to build the solution and refresh the entity definitions in the development environment so that the new field is visible when creating the DualWrite mapping.


#### Part 3 - Incorporate the new field into the DualWrite map
Now that we have the new field available in both systems, we need to map those fields as part of the DualWrite integration. The mappings provided by Microsoft are read-only, so to add our new field mapping we'll need to save a copy and modify that copy.

In the F&O environment, access the DualWrite maps by navigating to the 'Data management' workspace and click on the DualWrite tile:
![Screenshot of Finance & Operations data management workspace, highlighting Dual-Write tile](https://github.com/finopsfuntimes/ProjectOpsEmpFinDims/raw/main/ScreenShots/FODataMgmtWspc.JPG)

Once the list of maps has loaded, identify the 'Project Operations integration actuals' map `msdyn_actuals` by searching for 'actual' in the filter box in the top right of the screen:
![Screenshot of Finance & Operations system Dual-Write mappings list, filtered with text actual](https://github.com/finopsfuntimes/ProjectOpsEmpFinDims/raw/main/ScreenShots/FODualWriteMapsActual.JPG)

Click on the map to open the details page and if the map is currently in the 'running' state, stop it by clicking the 'stop' button in the top left of the screen. Once stopped, use the 'Save as' menu button to create the copy of this map, which we can then edit:
![Screenshot of Finance & Operations DualWrite mapping save as screen](https://github.com/finopsfuntimes/ProjectOpsEmpFinDims/raw/main/ScreenShots/FODualWriteMapSaveAs.JPG)

Click on the 'refresh tables' button to ensure that the custom field has been picked up in the table definition and then click on the 'add mapping' button to create the new field map. This will create a new entry at the bottom of the list, where you can click on the [none] link on each side of the map to find a select your email address custom field. The field is populated from the CE system and would not be modified in F&O, therefore click on the equals sign and change the sync direction from 'Bidirectional' to 'Common data service to Finance and Operations apps.' The screenshot shows an example of how this should look. Click on the 'save' button at the top of the screen and once saved, click on 'run' to ensure the map can start without errors.
![Screenshot of Finance & Operations DualWrite mapping, customised with bookable resource email field map](https://github.com/finopsfuntimes/ProjectOpsEmpFinDims/raw/main/ScreenShots/FODualWriteMapCustomised.JPG)

Even though the steps above were performed via the Finance & Operations user interface, the dual write maps are actually stored in the CE side, so the final step is to include the new custom DualWrite map as part of your solution in the Power Apps maker portal. Open your custom solution and click the 'Add existing' button, scroll down to pick 'Dual Write Entity Map' and then select the custom 'actuals' map that you created above to complete the solution.
![Screenshot of PowerApps maker portal adding DualWrite entity map to solution](https://github.com/finopsfuntimes/ProjectOpsEmpFinDims/raw/main/ScreenShots/MakeAddDWMap.JPG)

At this point, it would be advised to perform some testing to ensure that the email of a bookable resource is successfully passed over to the F&O system when a time entry is approved in the Project Operations model driven app. You can validate this using table browser in F&O by modifying the URL below with the base URL of the environment and the legal entity in which you are testing.

*Example URL to go here*

#### Part 4 - Extend F&O business logic to lookup default financial dimensions
There is a batch job in F&O which is responsible for creating the journal lines from the actuals records that were passed into F&O through DualWrite. The class containing this code is `ProjActualsImportIntegration`. This is an extension of another class, `ProjCDSActualsImportIntegration` which was used in previous versions of F&O when D365 PSA (Project Service Automation) was configured to integrate with it.

Much of the code which initialises the journal lines still sits inside the base class, and so we will extend that base class with additional business logic to lookup the default dimensions of the employee, based on the email address that was passed over with the 'actuals' record.

Returning to Visual Studio in the F&O development environment and the project created in part 2 of the customisation, we need to extend the class `ProjCDSActualsImportIntegration` and use the [chain of command approach](https://docs.microsoft.com/en-us/dynamics365/fin-ops-core/dev-itpro/extensibility/method-wrapping-coc) to allow us to append some code to the initJournalLine method, as shown in the screenshot below. We can then add a new method where we execute our custom business logic and call it after executing the standard logic.

*screenshot*

The code is wrapped in a check to ensure it only executes when the current legal entity is configured to run in Project Operations integrated mode and only if the record being processed actually contains a value in the email field. The steps we then have to go through to get the default dimensions are as follows and demonstrated in the code in the screenshot:

 - Use the email address to identify a user record
 - Get the worker record associated with that user record
 - Get the employment record for that worker in the current legal entity
 - Copy the financial dimensions onto the journal line

*screenshot*

The code sample above is good for demonstrating the steps and the individual data sources used to get to the required info but is inefficient because it makes multiple calls to the database. To address this, we will create a SQL view, which will allow us to quickly lookup the required info with a single query.

A SQL view is based on a query defined in the project, so we will go to add new object and choose query.

We then add the data sources, with their relations to the query definition.

Then we can create the SQL view and define it with the query we just created. A build will ne needed at this stage to synchronise the database and create the new view.

Finally, we can refactor our business logic to query our new view directly as below. You can grab the source code for the whole class from the repo here. *to be linked*

After another build, we are then ready to test the end to end solution to verify that any newly created journal lines are indeed picking up the default financial dimensions of the employee who performed the work!
 
