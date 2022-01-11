# Lab 3 :: Multi-tenancy

In Istio, a tenant is a group of users that share common access and priviledges for a set of deployed workloads. Tenants are like teams, and can be used to provide a level of isolation between different teams.

Istio supports three different types of tenancy models:
- Namespace tenancy
- Cluster tenancy
- Mesh tenancy

In this lab, we will dive into namespace tenancy where 3 teams each have their own hostname, deployed workloads, and Istio configuration while sharing the same cluster. What are running within each team should not impact any other team, unless there are service dependencies among the teams.

## Introduce our teams

Below are our A, B, C teams and their responsibilities:

Team A owns `web-api` service which is also exposed on the Istio ingress gateway as `web-api.istioinaction.io`. The `web-api` service is deployed in the web-api namespace. Team A is a producer for the `web-api.istioinaction.io` host name and a consumer of team B's `recommendation` service.

Team B owns the `recommendation` and `purchase-history` service. The recommendation service is consumed by the `web-api` service from Team A, along with some mobile applications thus it is also exposed on the Istio ingress gateway as recommendation.istioinaction.io. The `recommendation` service is deployed in the recommendation namespace and the `purchase-history` service is deployed in the purchase-history namespace. The `recommendation` service from team B is a producer for the `recommendation.istioinaction.io` host name and team A's `web-api` service. The `purchase-history` service is a private service where it is only produced/consumed for services within team B, where the `recommendation` service calls the `purchase-history` service. Nothing outside of team B should be able to view or call the `purchase-history` service.

Team C doesn't consume any other services, it produces the `ratings` service for some mobile applications via Istio ingress gateway as ratings.istioinaction.io but doesn't produce anything for team A or B to consume.

## Gateway and Virtual Service Isolation

VS delegation should be covered here.

Note: mention exposing ports on gateway as needed

### gateway merging


### virtual service delegation


### Global host name

Cover recommendation service using global hostname such as recommendation.istioinaction.io for client in the mesh and client out of the mesh.

## Service Isolation

### private service and public service

What if only 1 service is public and others are private services?

Sharing public services among teams?  To which team?

## Config Isolation

### Config isolation to specific ports

TODO: Talk about GM Workspace
