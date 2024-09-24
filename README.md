# Understanding Solution Layering and Publishers in the Power Platform

In the Power Platform, the concepts of solutions and solution layers are well understood and generally function as expected. This post isn't about introducing these concepts from scratch. Instead, we'll focus on the challenges you might face when dealing with different solution publishers.
Many newcomers to the platform start with the default solution publisher, which often seems unproblematic—until it’s too late. This blog post aims to shed light on how the choice of solution publisher can impact solution layering and dependency management within the Power Platform. We’ll also provide tips on how to avoid common pitfalls associated with these issues.

### Sample set-up
For simplicity's sake, we'll be using small solutions during this article to illustrate the concept of solution publishers. We'll be using 2 different solution publishers, Publisher A, and Publisher B. These publishers will be used in the development environment to create several different solutions, which will all contain only a set of Dataverse tables to illustrate the concepts.
![image](https://github.com/user-attachments/assets/f09c82fa-1105-462d-80a4-937ffc4871ba)

### Using the same publisher
The best-case scenario is having a single solution publisher across solutions, as this allows for transferring ownership of components between solutions and changing the layering of these solutions at a later point in time (we'll get to that). To show how this works, Solution A has a single table called "Table A", this solution is deployed to production. Solution A2 contains a small modification to that table, using solution segmentation, only the "Description" field of "Table A" is added to Solution A2 to only modify that field (changing the length of the field from 2000 to 1000). This modification is layered on top of Solution A in the production environment:
![image](https://github.com/user-attachments/assets/6c35b1a3-5261-498d-84e9-7d169a89b98e)

#### Removing solution A
Here the first weird behavior shows, the platform does not show any dependencies for solution A, while we know that solution A2 makes a modification to the table that lives in solution A. However, when trying to remove solution A, the platform will throw an error and not allow you to remove this. Imagine you really need to get rid of solution A, how would you do this? This can be achieved by moving the entire table A to solution A2:
1. Add Table A to solution A2, including all objects and table metadata.
2. Import a new managed version of solution A2 in production.
3. Remove solution A from production.

We have now effectively moved ownership of Table A to solution A2, as the table is now only present in managed solution A2, as can be seen below. This shows the benefits and flexibility that you have when using the same solution publisher:
- You can change "ownership" or the primary solution of a component any time you want.
- You can freely move objects between solutions with the same publisher.
- It is easy to understand layering and dependencies between solutions when they are all using the same publisher.

![image](https://github.com/user-attachments/assets/23deeea4-f908-4edc-950b-e4897f1faf39)

### Using different publishers
In the section above, the "happy flow" was described. Now, let's consider the same scenario, with one important difference. Instead of using Solution 2A as solution layer on top of Solution A, Solution B will be used. The set-up of Solution B is the same as before. Initially, this seems to be the same as before, showing 2 solution layers as expected. Again, we cannot remove Solution A because there is a dependency due to Solution B.
![image](https://github.com/user-attachments/assets/c3f6759e-afc1-4826-8bb9-9277699b21cf)

Now, let's try to replicate the second step, moving the entire table A to Solution B with the intent to remove Solution A from production. We follow the same steps as above:
1. Add Table A to solution B, including all objects and metadata.
2. Import a new managed version of solution B in production.
3. Remove solution A from production.
The first two steps work fine, however, when trying to remove solution A, we are blocked due to the difference in solution publishers. So in the current state, we cannot remove solution A from the production environment. How could we proceed now?

As the issue seems to be related to Table A, you might want to try removing table A from solution A in dev, pushing an upgrade of solution A to production and then repeat the import of solution B. However, you will find that you get the exact same error, and the upgrade will not import successfully. 

Another thought could be to use the solution A2 we used before, as this also contains Table A, as these solutions both use publisher A, this should work right? And indeed, we can import solution A2, however due to solution layering, solution A2 will be placed on top of the stack, layering over solution B, with sits on top of solution A. This again means that we cannot uninstall solution A, as solution B (with a different publisher) depends on this solution.  So, we need to remove B and reimport it again to make sure it sits on top of the stack, after which solution A can be removed. Note that this works in our current simple scenario but is not feasible for actual projects as solution B will most likely contain objects that should not be removed from the environment.

### What now?
As the section above showed, using different solution publishers complicates dependencies and layering within the platform. In short, the main difficulties are:
- The base layer solution is the owner of the component. The solution at the bottom of the layers is the owner of the component, keep this in mind as this influences the points below.
- It's possible to move the ownership of a component from one solution to another within the same publisher, but not across publishers. This is shown in scenario 2, uninstalling solution A basically means that solution B will become the owner of the component as there is no other solution containing the component anymore. And as solution B has a different publisher, this is not possible.
- Once you introduce a publisher for a component in a managed solution, you can’t change the publisher for the component. In our example, the moment we created table A, the publisher was set, and this cannot be changed anymore. In theory, you could change this in the solution files/xml, but this effectively means creating a new table in Dataverse as the publisher prefix is used in the system name, so this would not be workable on an application that already contains data or is on production.
- Solution publishers cannot be changed during solution upgrade. When a solution is already present in an environment, you cannot change the solution publisher of that solution during an upgrade. The only way to do that is by removing the solution completely and reinstalling, but this means losing data.

The above examples highlight the importance of using the same publisher for related solutions, but what if you already have different publishers and are already running into these issues? The platform does not support changing the publisher for components, and even if you try to do this via xml and succeed, this means that you will be losing data. So, the best you can do is restructure the solutions as much as possible and start new developments using the proper publishing set-up. To restructure the solutions and negate the above points, think of the following steps:
- Create an overview of all solutions and publishers you are using, as well as the layer order that applies to your downstream environments.
- Identify the "cross-publisher" layering for components, in the example above, identify the scenario where solution B (publisher B) contains a components of publisher A.
- For these cross-publisher layers, try to clean those up by moving the customization to a new solution with the same publisher as the component, after which you can remove the customization from the other solution (solution B in this case). 
- Introduce a proper solution publisher for any new developments.
In our sample, these steps will result in the following set-up:
- Solution A will contain Table A.
- Solution B will contain solution B related components, but will no longer contain Table A.
- The customization solution B did on Table A is now moved to Solution A - Customizations, which has Publisher A as the publisher.
- For new customizations, we can decide to always use Publisher A, or to create a new one that we will use from now on.

This means that solution B has no more dependency on publisher A, solution A can now easily be removed if necessary and the new base layer will then be Solution A - Customizations, as changing ownership between solutions with the same publisher is allowed. Of course, the ideal scenario would be to have a single publisher, but this is not possible when already having this existing set-up.




