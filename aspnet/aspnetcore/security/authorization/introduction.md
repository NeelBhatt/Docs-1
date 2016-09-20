---
uid: security/authorization/introduction
---
<a name=security-authorization-introduction></a>

  # Introduction

Authorization refers to the process that determines what a user is able to do. For example user Adam may be able to create a document library, add documents, edit documents and delete them. User Bob may only be authorized to read documents in a single library.

Authorization is orthogonal and independent from authentication, which is the process of ascertaining who a user is. Authentication may create one or more identities for the current user.

  ## Authorization Types

In ASP.NET Core authorization now provides simple declarative [role](roles.md#security-authorization-role-based.md) and a [richer policy based](policies.md#security-authorization-policies-based.md) model where authorization is expressed in requirements and handlers evaluate a users claims against requirements. Imperative checks can be based on simple policies or polices which evaluate both the user identity and properties of the resource that the user is attempting to access.

  ## Namespaces

Authorization components, including the [AuthorizeAttribute](http://docs.asp.net/projects/api/en/latest/autoapi/Microsoft/AspNetCore/Authorization/AuthorizeAttribute/index.html.md#Microsoft.AspNetCore.Authorization.AuthorizeAttribute.md) and [AllowAnonymousAttribute](http://docs.asp.net/projects/api/en/latest/autoapi/Microsoft/AspNetCore/Authorization/AllowAnonymousAttribute/index.html.md#Microsoft.AspNetCore.Authorization.AllowAnonymousAttribute.md) attributes are found in the [Microsoft.AspNetCore.Authorization](http://docs.asp.net/projects/api/en/latest/autoapi/Microsoft/AspNetCore/Authorization/index.html.md#Microsoft.AspNetCore.Authorization.md) namespace.
