---
title: User Types and Roles in an Activity-based Permission Systems
published: true
description: How to think about a role-based activity check permissions system
tags: permissions, authorization, access control
date: 2019-01-04
---

_Decoupling User Types from User Roles in Activity-based Permission Systems_ 


#### TLDR;
Lets say we are developing an app. In our app we should have lots of granular, atomic permissions. We should support roles (groupings of permissions) but as something defined by admin-users for their organization, and so be 100% arbitrary to us. These roles should exist within Profile types, which _are_ defined by us, and are used to know which "type" of user we are dealing with. A user can have multiple roles within and across multiple profiles, but can only be acting as one profile at a time. Our app code, when dealing with a user, can check their Profile (ie `user.is(Profile)` to answer the question "where should we send them?") or perform permission checks (ie, `user.can(Operation, Object)` to answer the question "what should they be able to do/see when they get there?").


## Concepts

**Permissions**
- A Permission is a granular activity-based access control.
- Permissions are specified by the development team.

**Profiles**
- A Profile is a supported user type in the app.
- Profiles are specified by the development team.
- Profiles (user types) are used to determined _where_ a user should go based. They correspond with tracks built the app (eg, which dashboard to go to).
- For each Profile an organization creates their own roles.
- A user may have roles in more than one profile.

**Roles**
- A Role is a group of permissions for a Profile.
- Roles are not specified by the development team and are completely arbitrary, and instead are specified by privileged users for their organization.
- A user may have more than one role per profile.

**User**
- A user has exactly one active profile (which has at least one role), but can switch profiles.
- A user may have roles in multiple organizations.

**Organization**
- An Organization is a grouping entity with an internal hierarchy.
- Roles exist for a Profile for an Organization.


Profiles exist to answer questions related to which "type" of user the app is dealing with. Permissions exist to answer questions related to what access the user can have. (EG, Profile: "Should the user be routed to the admin or affiliate dashboard?" vs Permission: "What should they see and be able to edit once they are routed?") Roles exist to in service of making permissions manageable.


Without profiles, you end up with a set of faux permissions - "permissions" which really are just interested in what type the user is (eg, canSeeAdminDashboard and canSeeAffiliateDashboard are really just questions about what is the type of user using the app).

Without roles, every user is a one-off set of permissions. For example, If you're creating multiple users for managing invoices, you'd have to re-select exactly the same permissions for each user, including months later after you hire another person to manage invoices. Or, suppose for example a new invoice feature comes out later. You'd have to remember which users are doing invoice work, and update each of them with this new permission.

Without organizations, roles would have to be global and governed by the development team. Instead, each organization in the system can configure their own roles for their applicable profiles - choosing permissions defined by the development team for that profile.


## Schema

```
operation               # actions types in the system (read, edit, create, etc)
- id

object                  # entity types to be acted upon (invoices, customers, etc)
- id

permission              # atomic abilities in the app
- id
- operation_id
- object_id

role                    # an organization's group of permissions for a profile
- id
- value
- label
- profile_id
- organization_id

role_permission         # specifies which permissions are in a role
- id
- role_id
- permission_id

profile                 # a supported user identity type in the app
- id
- value
- label

profile_permission      # specifies which profiles a permission is valid for
- id
- permission_id
- profile_id

organization
- id

user
- id
- active_profile_id

user_role               # specifies the roles a user has
- user_id
- role_id

# There would be no `user_profile` table, as access to a profile is a permission
```

## Code

`User`
- `is(Profile)` - returns whether the user is actively acting as a certain profile type (answering questions like "where should we send this user?")
- `can(Operation, Object, Organization?)` - iterates through the user's roles (for a specific organization if supplied) for their active profile, to identify if they have permission (answering questions like "what resources should the user be able to see/change?")
- `switchProfile(Profile)` - changes the user's active profile



## Making it Less Abstract: An Example

Let's say you and I create an app for lawn care businesses.
- _Lawn Care Owners_ log in to create clients, create employees, assign jobs to employees, send invoices to their clients, and write monthly newsletters about lawn care tips to their clients.
- _Lawn Care Workers_ log in to see their schedule.
- _Clients_ log in to pay their invoices, and see past invoices.
- As part of an affiliate-system, _Home Owner Associations_ can log in to see clients in their neighborhoods.
- _Product Marketing Companies_ can log in to post ads to this app, since its free to the business owner.
- Finally, _Internal Staff_ (you and I and our employees) can log in to see data and metrics and charts and stuff.

Thus, we create these six Profiles:
- Lawn Care Administrator
- Lawn Care Worker
- Client
- Homeowner Association Rep
- Brand Rep
- Internal Staff

While, each profile is known and has special meaningful to us, roles are all user-generated and are never referenced or known in our code.

Let's start by looking at the first three.

In the beginning, the owner makes one role within each profile, and assigns it to himself. As the company grows, he hires employees for each of these, and soon has multiple employees type: Just in administration, a manager to schedule clients and assign lawn care workers to them; a bookkeeper to see the paid/unpaid finances; a marketing intern to write up the newsletters. Similarly, there arises a hierarchy within his growing crew of lawn care workers, constraining permissions for new guys and allowing the full breadth of permissions for his team lead. Lastly, as commercial clients are taken on, a role with different permissions is created.

```
Administrator
 - Scheduler
 - Bookkeeper
 - Marketer

Lawn Care Worker
 - Entry Level
 - Experienced
 - Team Lead

Client
 - Residential
 - Commercial
```

In reality, these roles would be not be focused on the "who" and instead on the "what". For example, if the difference between residential and commercial as defined above is really about the difference in how the owner allows the client to pay, it would make more sense to have a "trusted_payer" permission, which allows the client to not have their credit card on file and be invoiced instead, etc. Then, when the owners favorite customer reads about credit card scams and asks to be invoiced rather than his card on file, that user's account can be meaningfully updated. Remember, a user can have multiple roles, it's not an either or. Permissions are just about "grouping" permissions for convenience and consistency! They're _not_ about creating fully descriptive categories. (Composition vs Category)

The next type of user who will be logging in to our app is a _Homeowner Association Rep_. They decide to create a role with limited permissions and allow the HOA members to see some of the metrics for this new source of income for the HOA.

```
Homeowner Association Rep
 - President
 - Member
```

The next type of user is a _Brand Rep_. They log in to see ad metrics for their campaigns, and created a role with limited permissions for their intern.

```
Brand Rep
 - Full
 - Intern
```

The last type of user is an _Internal Staff_ user. These are your and my employees, and they log in to see and configure data.

```
Internal Staff
 - Superuser
 - Customer Support
 - Marketer
```

Each of these roles under the six above profiles are specific to the "organization" that made them. In the case of the fist three profiles, those roles are part of the lawn care business's organization; for the HOA rep profile, the hoa organization; for the brand rep profile, the brand organization; and finally for our employers, an internal staff organization.

This might sound odd, that these organizations aren't parallel. That's true if an organization includes the business or hoa or brand specific details, but our organizations do not. There would be a separate "business"/"hoa"/"brand" table for this data. An organization is just an entity used for permissions, and not coupled with a business, or hoa, or brand, or tenant, or conglomerate, or whatever-grouping-you-may-use.


A user can perform the actions within an organization according to their roles (which are organization-aware).

Lets say a user is:
- a client who uses both Tom's Lawn Care (for weekly mowing) and Jack's Landscaping (for less frequent landscaping tune-ups)
- the president of a Home Owner's Association, and use the affiliate referral program to raise money for their HOA (Blue Meadows)
- a marketing employee for our app company, using the staff app.

This user account is associated with four different organizations: Tom's, Jack's, Blue Meadows, and Internal Staff. It is associated with three different profiles: client, homeowner association rep, and internal staff.

When this user logs in, they are presented with their profiles. If they choose Internal Staff, they see can see metrics and data to better market our app. If they choose Homeowner Association Rep, they see a dashboard with their referral kickbacks. If they choose Client, they see the invoices from Tom and Jack, and since Jack has given them more permissions, for his they can see line items of his costs. One day, Tom gets spooked by the internet and removes all client permissions, and so only Jack's show up here.


Thus, a user may have multiple profiles with multiple roles across multiple organizations - but our app code only cares about profiles, organizations, and permissions (not roles). If a 

It is important to note, roles within an organization within a profile "add up" - even if the permission is missing in one role in a profile, if it is present in another, the user has that permission.


### Example UIs

Super rough demo example UI of permissions when a business owner is associating a new user with his organization:
![Permissions](https://thepracticaldev.s3.amazonaws.com/i/xpjvjly5q01506w13tqb.png)

Super rough demo example of a business owner adding a new role for his organization:
![Add New Profile](https://thepracticaldev.s3.amazonaws.com/i/ysy353ke6hqwqpc6002f.png)


When a user with more than one profile logs in, the app must enforce only allowing the user to act as one profile (and so making the user select which profile, or defaulting them to the last profile, etc).



# Inspiration / Better Articles:

[Role-Based Access Control](https://csrc.nist.gov/CSRC/media/Presentations/Role-based-Access-Control-an-Overview/images-media/alvarez.pdf) by the National Institute of Standards and Technology
[Role Based Access Control](https://www.enterpriseready.io/square/role-based-access-control/) by Square
[Don’t Do Role-Based Authorization Checks; Do Activity-Based Checks](https://lostechies.com/derickbailey/2011/05/24/dont-do-role-based-authorization-checks-do-activity-based-checks/) by Derick Bailey
