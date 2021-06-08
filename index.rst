:tocdepth: 1

.. sectnum::

Abstract
========

The Interim Data Facility (IDF) is hosted at Google using Google Cloud Platform (GCP).
Administrators of the IDF must be able to authenticate to the Google infrastructure to debug problems and must have a mechanism to make changes to the IDF configuration.
This document outlines the identity, authentication, and authorization model used for the IDF and discusses the security goals that it is intended to address.
It also discusses some non-default configuration options for the GCP organization that are set for security reasons.

This document is heavily based on *Rubin Observatory GCP Onboarding* written by Burwood Group, Inc., for Rubin Observatory.
It summarizes the configuration information for identity, authentication, authorization, and security configuration from that document, with modifications for subsequent configuration changes.

Organization
============

The IDF is managed through the lsst.cloud Google Workplaces account, with most services (such as Gmail and Google Docs) disabled.
Authentication is managed through the associated lsst.cloud Google Cloud Identity domain.
All users with administrative access to the infrastructure hosting the IDF must authenticate with lsst.cloud identities.

This Google Workplaces account corresponds to the lsst.cloud GCP organization.
Within that organization is a set of folders corresponding to high-level service areas of the IDF, plus a Scratch folder for experiments and a Shared Services folder for infrastructure shared by all projects.
Within each folder is a set of projects that contain GCP resources such as Google Kubernetes Engine clusters or Google Compute Engine nodes.
In general, the projects are divided into a dev project, an integration (int) project, and a production (prod or stable) project.

Terraform
---------

The GCP configuration is managed by Terraform in the `lsst/idf_deploy <https://github.com/lsst/idf_deploy>`__ GitHub project.
The configuration expressed in Terraform is automatically deployed using GitHub Actions when merged to the master branch of the repository.
The Google service account keys for Terraform are stored as secrets in the GitHub repository so that they are accessible by GitHub Actions.

Terraform uses a service account in the rubin-automation-prod project in the Shared Services folder to authenticate to GCP, and stores its state file in a Google Cloud Storage (GCS) bucket in that same project.

As much as possible, all GCP configuration is done via Terraform.
Exceptions include legacy projects (the sqre project in the Production folder under the SQuaRE folder, for example) and experiments that will be torn down later.

Super admins
------------

A Google Workspace has "super admins" who have irrevocable administrative permissions to manage the Google Workspace account, such as setting permissions and passwords for any user.
The IDF follows the `Google best practices for super admins <https://cloud.google.com/resource-manager/docs/super-admin-best-practices>`__.
This, among other things, means that super admin accounts are not used for day-to-day work and should not be a member of any of the Google Groups that control access to GCP or other resources.
They should be used only to set up new permissions or debug or fix problems that require an administrator.

Only super admins have access to the admin console for the lsst.cloud Google Workspace.

Following the standard convention, super admin accounts append ``-admin`` to the user's normal username.
For example, the super admin account of a user whose regular account is ``rra@lsst.cloud`` will be ``rra-admin@lsst.cloud``.

Authentication
==============

Access to Google Cloud Platform for the IDF is restricted to lsst.cloud accounts plus some accounts from consultants and partners.
All Rubin Observatory staff with access to the infrastructure must use lsst.cloud accounts, not personal accounts, lsst.org accounts, or anything else.
(In other words, Domain Restricted Sharing is enabled and only lsst.cloud and consultant domains are whitelisted.)
This allows enforcement of authentication policies for all users.

User authentication
-------------------

2-Step Verification (Google's name for two-factor authentication) is required for all lsst.cloud accounts.
Currently, all methods are allowed, including SMS and phone call.
However, use of either a security key (if one already has one; Rubin Observatory does not provide them) or Google Authenticator or a similar cellphone app is strongly encouraged.
SMS and phone calls are less secure and are discouraged.

Account recovery is disabled for all users.
Users who lose their passwords or are otherwise locked out of their accounts should contact a super admin.

Use of a password manager and a randomized password for the lsst.cloud account password is strongly encouraged.

Service accounts
----------------

Service accounts are created within the project most directly related to what the service account does, or in a project in the Shared Services folder for cross-project infrastructure such as Terraform.

IAM grants for the default service accounts are disabled.
The IDF should not use the default service accounts to make Google API calls and instead create a new service account with only the required permissions.

One of the most common attacks on cloud infrastructure is theft of the key of a service account that was improperly stored and then use of that key to break into other services.
Key creation for service accounts is therefore disabled by policy.
Service accounts should normally be Google-managed and exposed to the relevant GKE or GCE instance using Google's facilities.
Google will then automatically rotate the keys, significantly reducing the risk of compromise.
Exceptions to this policy will be made if a service account key is required because the service using it cannot use Google-managed keys.
(Terraform, for example, has to have a key because it needs to be stored in GitHub Actions.)

Service accounts should be created and assigned permissions using Terraform.

Authorization
=============

Authorization of human users is done via Google Groups.
Users are granted membership in groups by super admins via the admin console, and then groups are granted access to GCP resources using Terraform.

Service accounts are granted IAM permissions directly using Terraform.

The group structure follows some naming conventions (all of the following are ``@lsst.cloud``):

gcp-organization-administrators
    Access to manipulate the folder structure inside the GCP organization, create new projects, and related tasks.

gcp-org-network-admin
    Access to manage shared VPCs.

gcp-org-security-admin
    Access to define IAM policies, security scans, firewall rules, and SSL certificates.

gcp-billing-administrators
    The billing account administrator and creator.

gcp-<folder>-administrators
    Grants full administrative access to the GCP projects contained in a given folder.

gcp-<folder>-gke-cluster-admins
    Grants full GKE administrative access (but not access to other GCP resources) for the projects contained in a given folder.

gcp-<folder>-gke-developer
    Grants full GKE administrative access to the dev (but not the production) GKE project inside a given folder.

There are some other groups to control access to specific functions, such as billing reports or configuration stored in the Shared Services folder.

Databases
=========

Public IP access to Cloud SQL instances is disabled by organization policy to prevent accidentally exposing Cloud SQL instances to the public Internet.

As a general rule, access to Cloud SQL instances should use the `Google Cloud SQL Auth Proxy <https://cloud.google.com/sql/docs/postgres/sql-proxy>`__.
For Kubernetes services, this should run as a sidecar container using a Kubernetes service account that is bound to an IAM service account with the appropriate IAM roles to connect to the Cloud SQL instance.
If a particular use case cannot use the Cloud SQL Auth Proxy, it can get an exception to allow direct connection to the database, but it should still be via private IP, not public IP.

Be aware that the Cloud SQL Auth Proxy does not replace database authentication.
The Cloud SQL instance will still need configured users with passwords, and those passwords will have to be shared with the services that use that instance.
