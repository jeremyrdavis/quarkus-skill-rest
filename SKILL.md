---
name: quarkus-rest
description: >
  Use when writing or modifying JAX-RS code in a Quarkus app — building Response objects, choosing HTTP status
  codes, writing ExceptionMappers, applying Bean Validation (@Valid, @NotNull, @Size, @Email), or handling
  content negotiation. Trigger on @Path, @GET / @POST / @PUT / @DELETE, MediaType, jakarta.ws.rs.core.Response,
  or @Provider implementing ExceptionMapper. Excludes resource layer placement (which package the resource lives
  in) and DTO design — those are architectural concerns covered by your team's standards.
---

# Quarkus REST Skill

Conventions for writing JAX-RS endpoints in Quarkus. Resources are thin: validate input, delegate to an
application service, translate the result into an HTTP response. They never contain business logic.

---

## When to Use

- Writing or editing a JAX-RS resource class (`@Path` annotated).
- Choosing the right HTTP status code or constructing a `Response` object.
- Wiring up Bean Validation on request parameters or bodies.
- Adding an `ExceptionMapper` to translate a domain exception into a typed response.
- Reviewing a diff that returns an entity directly, throws raw `WebApplicationException`, or builds responses
  inconsistently across methods.

**Out of scope**: which package the resource lives in (architectural layering — covered by your team's
conventions), DTO / record shape design, authentication / authorization wiring (Quarkus Security, separate
concern).

---

## Core Rules

1. **Always return `jakarta.ws.rs.core.Response`.** Never return the entity / DTO directly from a resource method.
   Returning `Response` makes the status code and headers explicit and consistent.
2. **Inject an application service; delegate everything.** Resources orchestrate HTTP, not business logic. No
   repository access, no transaction management, no domain validation in resource methods.
3. **Use `@Valid` on bodies, `@NotNull` / `@NotBlank` / `@Size` / `@Email` on params and DTO fields.** Bean
   Validation triggers a 400 automatically — don't write null checks by hand.
4. **Pick the status code from the table below.** Don't default to 200 for everything; `201` for creates with a
   `Location` header, `204` for no-content updates and deletes, `404` for missing resources.
5. **Map domain exceptions to HTTP via `ExceptionMapper<E>`.** Don't `try/catch` domain exceptions in every resource
   method. Define one mapper per exception type; the resource code stays linear.
6. **Endpoint paths are consistent across the service.** Whether your project uses `/orders` or `/api/orders`,
   pick one prefix convention and apply it everywhere. Don't mix prefixes within the same service — frontend
   bookmarks, monitoring dashboards, rate-limit rules, and customer integrations will latch onto whichever
   path ships first.
7. **Log at the boundary, not on the happy path.** Resources delegate; the application service already logs
   state changes. Resources log only on inbound validation failures or 5xx mapping. (See the companion
   `quarkus-logging` skill for log call-site rules.)

---

## Canonical Example

```java
package com.example.orders.interfaces.rest;

import com.example.orders.application.OrderApplicationService;
import com.example.orders.application.OrderDTO;
import com.example.orders.application.PlaceOrderRequest;
import io.quarkus.logging.Log;
import jakarta.inject.Inject;
import jakarta.validation.Valid;
import jakarta.validation.constraints.NotNull;
import jakarta.ws.rs.*;
import jakarta.ws.rs.core.MediaType;
import jakarta.ws.rs.core.Response;
import jakarta.ws.rs.core.UriInfo;
import jakarta.ws.rs.core.Context;
import java.util.Collection;
import java.util.Optional;

@Path("/orders")
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
public class OrdersResource {

    @Inject
    OrderApplicationService orders;

    @GET
    public Response listOrders() {
        Collection<OrderDTO> result = orders.findAll();
        return Response.ok(result).build();
    }

    @GET
    @Path("/{id}")
    public Response getOrder(@PathParam("id") @NotNull Long id) {
        Optional<OrderDTO> result = orders.findById(id);
        return result
            .map(dto -> Response.ok(dto).build())
            .orElseGet(() -> Response.status(Response.Status.NOT_FOUND).build());
    }

    @POST
    public Response placeOrder(@Valid PlaceOrderRequest request, @Context UriInfo uriInfo) {
        Long newId = orders.placeOrder(request);
        var location = uriInfo.getAbsolutePathBuilder().path(newId.toString()).build();
        return Response.created(location).build();
    }

    @DELETE
    @Path("/{id}")
    public Response cancelOrder(@PathParam("id") @NotNull Long id) {
        orders.cancel(id);
        return Response.noContent().build();
    }
}
```

Companion `ExceptionMapper` for a domain rejection:

```java
package com.example.orders.interfaces.rest;

import com.example.orders.domain.OrderNotFoundException;
import io.quarkus.logging.Log;
import jakarta.ws.rs.core.Response;
import jakarta.ws.rs.ext.ExceptionMapper;
import jakarta.ws.rs.ext.Provider;

@Provider
public class OrderNotFoundMapper implements ExceptionMapper<OrderNotFoundException> {
    @Override
    public Response toResponse(OrderNotFoundException e) {
        Log.debugf("order not found id=%s", e.orderId());
        return Response.status(Response.Status.NOT_FOUND)
            .entity(new ErrorBody("order_not_found", e.getMessage()))
            .build();
    }

    public record ErrorBody(String code, String message) {}
}
```

What this demonstrates:

- Every method returns `Response`. Status codes are explicit (`ok`, `created`, `noContent`, `NOT_FOUND`).
- Resource is thin: no logic, no repositories, no transactions. Delegates to `OrderApplicationService`.
- Bean Validation on params (`@NotNull`) and bodies (`@Valid`) — the 400 response is automatic.
- 201 on create includes a `Location` header built from `UriInfo`.
- Domain exceptions are mapped centrally via `@Provider`, not caught inline.
- Logging follows `quarkus-logging`: `*f` variant, identifiers only, no PII.

---

## Anti-patterns

| Don't | Why it's wrong | Fix |
|---|---|---|
| `public OrderDTO getOrder(...) { ... }` (returning the DTO directly) | Status code is implicit (always 200). Can't return 404, 204, 201 with a Location, or set custom headers. | Return `Response` and call `Response.ok(dto).build()`. |
| Throwing `WebApplicationException(404)` inline | Couples HTTP status to the resource layer in the wrong place; spreads the same translation across every method. | Define `OrderNotFoundException` in the domain; map it once via `@Provider ExceptionMapper`. |
| `try { ... } catch (DomainException e) { return Response.status(400)... }` in every method | Repetitive, easy to forget, and mixes HTTP concerns into the resource body. | Use an `ExceptionMapper` per exception type. |
| `if (request.customerId == null) return Response.status(400)...` | Hand-rolled validation. Bean Validation already does this. | `@Valid` on the body parameter; `@NotNull` on the field in the request record. |
| `@POST` returns `Response.ok(newId)` | Wrong status; creates should be `201 Created` with a `Location` header pointing to the new resource. | `Response.created(uriInfo.getAbsolutePathBuilder().path(id.toString()).build()).build()` |
| `@DELETE` returns the deleted entity | Wrong status; deletes are `204 No Content`. | `Response.noContent().build()`. |
| Logging the full request body in the resource | Likely contains PII; resource is the wrong layer to log domain state changes. | Log identifiers only, and let the application service log state transitions. |
| Path inconsistent with the rest of the service (e.g. mixing `/orders` and `/api/orders`) | Two paths to the same resource is technical debt that compounds. | Pick one prefix convention and apply it everywhere. |

---

## Excuse / Reality

When you catch yourself reasoning around the rules above, look here before you type. The left column is verbatim — what you'll actually say in your head or in Slack. The right column is what defeats it.

| Excuse | Reality |
|---|---|
| "Option B is good enough — at least it returns `Response`." | `Response.ok(entity)` still serializes the JPA proxy. The frontend receives a stack trace mid-response (lazy fields), not a useful payload. Partial compliance is non-compliance. |
| "Just expose the entity at `/api/orders/{id}` so the frontend has all the fields. We'll prettify later." | Lazy-loaded fields throw `LazyInitializationException` once the response leaves the transaction. "All the fields" arrives as a 500. Map to a DTO inside the application service. |
| "Two paths — `/orders` and `/api/orders` — won't matter; we'll standardize next sprint." | Frontend bookmarks, monitoring dashboards, rate-limit rules, and customer integrations latch onto whichever path ships first. Two paths is forever. Pick one the first time. |
| "I'll wrap a `try/catch` in this resource just for this one exception." | One `try/catch` becomes the template. Three resources later, HTTP-status decisions are scattered across every method instead of consolidated in one `ExceptionMapper`. |
| "It's a quick demo endpoint; it doesn't need DTO mapping." | Demos go to production. The "quick demo endpoint" is the one the executive remembers and the team forgets to refactor. |

---

## Quick Reference

### Status code chooser

| Situation | Status | `Response` builder |
|---|---|---|
| Successful read with body | 200 OK | `Response.ok(body).build()` |
| Successful create | 201 Created | `Response.created(location).build()` |
| Successful update with no body | 204 No Content | `Response.noContent().build()` |
| Successful delete | 204 No Content | `Response.noContent().build()` |
| Validation failure | 400 Bad Request | Bean Validation produces this automatically |
| Authentication required | 401 Unauthorized | Quarkus Security; out of scope here |
| Authenticated but forbidden | 403 Forbidden | Quarkus Security |
| Resource missing | 404 Not Found | `Response.status(Response.Status.NOT_FOUND).build()` |
| Method not allowed for path | 405 | Framework auto-emits |
| State conflict (e.g. version mismatch) | 409 Conflict | `Response.status(Response.Status.CONFLICT).build()` |
| Body parses but violates a domain invariant | 422 Unprocessable Entity | `Response.status(422).entity(error).build()` |
| Unhandled server error | 500 | `ExceptionMapper<Throwable>` (or framework default) |

### Bean Validation cheat sheet

| Constraint | Use on |
|---|---|
| `@NotNull` | Path / query params and DTO fields that must be present |
| `@NotBlank` | String fields that must be non-null and non-empty |
| `@Size(min=, max=)` | Strings, collections |
| `@Email` | Email-shaped string fields |
| `@Positive` / `@PositiveOrZero` | Numbers |
| `@Pattern(regexp=...)` | Custom string formats |
| `@Valid` (on body parameter) | Cascade validation into the DTO |

### Resource method skeleton

```java
@<HTTP-VERB>
@Path("/...")    // optional
public Response <verb><Noun>(<params with constraints>) {
    <result> = applicationService.<method>(<args>);
    return Response.<status>(<result>).build();
}
```

If the method needs more than 5 lines of logic, the logic belongs in the application service, not the resource.
