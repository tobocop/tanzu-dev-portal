---
draft: false
date: 2021-05-05T10:40:17-06:00
tags: ["spring","testing", "spring security"]
title: "Spring Route Authorization Testing"
description: >
    Dynamic testing of your Spring Security route authorizations
date: "2021-05-12"
topics:
- Spring
- Testing
tags:
- Spring
- Spring Security
- Spring Testing
# Author(s)
team: 
- Toby Rumans
---

Spring Security provides us with the ability to lock down routes in our applications to particular roles. However, testing route authorizations often ends up feeling manual and prone to human error. What this approach proposes is a few dynamic tests that ensure your route authorization configuration is always up to date and has 100% coverage.

Code Examples: <https://github.com/tobocop/spring-route-authorization-testing>

_Note_: The solutions provided were coded for `spring-boot-starter-security:2.4.4` and junit 5.

## Solution - Kotlin

Kotlin code example: <https://github.com/tobocop/spring-route-authorization-testing/tree/main/kotlin>

Here is the finished test class we'll be walking through:

```kotlin
@SpringBootTest
@AutoConfigureMockMvc
internal class WebSecurityConfigTest {

    val routeAuthSpecs: Set<RouteAuthSpec> = setOf(
        RouteAuthSpec("/user/{id}", HttpMethod.GET, Access.Unauthenticated),
        RouteAuthSpec("/user/{id}", HttpMethod.PUT, Access.AnyRole(Role.ADMIN, Role.BASIC)),
        RouteAuthSpec("/user/{id}", HttpMethod.PATCH, Access.AnyRole(Role.ADMIN, Role.BASIC)),
        RouteAuthSpec("/user", HttpMethod.POST, Access.AnyRole(Role.ADMIN)),
        RouteAuthSpec("/user/{id}", HttpMethod.DELETE, Access.AnyRole(Role.ADMIN)),
        RouteAuthSpec("/some-route-without-correct-auth-to-show-failure", HttpMethod.POST, Access.AnyRole(Role.ADMIN)),
        RouteAuthSpec("/some-route-not-implemented-to-show-failure", HttpMethod.GET, Access.Unauthenticated),
    )

    @Autowired
    private lateinit var mockMvc: MockMvc

    @Autowired
    lateinit var requestMapping: RequestMappingHandlerMapping

    @Test
    fun `tests every route and no non-existent routes`() {
        val allRoutes = requestMapping.handlerMethods.keys
            .filter { it.patternsCondition != null }
            .filterNot { it.patternsCondition!!.patterns.contains("/error") }
            .flatMap { requestMappingInfo ->
                val method = requestMappingInfo.methodsCondition.methods.first()
                val routes = requestMappingInfo.patternsCondition!!.patterns
                routes.map { route -> "$method $route" }
            }
        val testedRoutes = routeAuthSpecs.map { spec ->
            "${spec.verb} ${spec.route}"
        }

        val untestedRoutes: Set<String> = allRoutes.subtract(testedRoutes)
        val nonexistentRoutes = testedRoutes.minus("GET /index").subtract(allRoutes)

        assertAll(
            { assertTrue(untestedRoutes.isEmpty(), String.format("The following routes are untested: %s", untestedRoutes)) },
            { assertTrue(nonexistentRoutes.isEmpty(), String.format("Tests are defined for the following nonexistent routes: %s", nonexistentRoutes)) }
        )
    }

    @TestFactory
    fun `all routes have correct authorization when unauthenticated`(): List<DynamicTest> =
        routeAuthSpecs.map { spec ->
            dynamicTest("Unauthenticated [${spec.verb}] ${spec.route}") {
                assertCorrectUnauthenticated(spec)
            }
        }

    @TestFactory
    fun `all routes have correct authorization when authenticated`(): List<DynamicTest> =
        Role.values().flatMap { role ->
            routeAuthSpecs.map { spec ->
                dynamicTest("$role [${spec.verb}] ${spec.route}") {
                    assertCorrectAuthz(role, spec)
                }
            }
        }

    private fun assertCorrectUnauthenticated(spec: RouteAuthSpec) {
        val result = mockMvc.perform(spec.request).andReturn()

        assertNotEquals(HttpStatus.METHOD_NOT_ALLOWED.value(), result.response.status,
            "Method ${spec.verb} does not exist for route ${spec.route}")

        when (spec.access) {
            is Access.Unauthenticated -> {
                assertNotEquals(HttpStatus.UNAUTHORIZED.value(), result.response.status,
                    "Expected ${spec.verb} ${spec.route} not to require authentication")
            }
            else -> {
                assertEquals(HttpStatus.UNAUTHORIZED.value(), result.response.status,
                    "Expected ${spec.verb} ${spec.route} to require authentication.")
            }
        }
    }

    private fun assertCorrectAuthz(role: Role, spec: RouteAuthSpec) {
        val result = mockMvc.perform(
            spec.request.with(
                SecurityMockMvcRequestPostProcessors
                    .user("automation-user")
                    .roles(role.toString())
            )
        ).andReturn()

        when (spec.access) {
            is Access.AnyRole -> {
                if (spec.access.allowedForRole(role)) {
                    assertNotEquals(HttpStatus.FORBIDDEN.value(), result.response.status,
                        "Expected role $role to be PERMITTED to ${spec.verb} ${spec.route}")
                } else {
                    assertEquals(HttpStatus.FORBIDDEN.value(), result.response.status,
                        "Expected role $role to be FORBIDDEN to ${spec.verb} ${spec.route}")
                }
            }
            is Access.Unauthenticated -> {
                assertNotEquals(HttpStatus.FORBIDDEN.value(), result.response.status,
                    "Expected role $role to be PERMITTED to ${spec.verb} ${spec.route}")
            }
        }
    }
}
```

### Solution - Kotlin - Setup

The basic premise is to use a combination of the `RequestMappingHandlerMapping` object from Spring Web and a few `@TestFactory` tests to ensure every route is tested with every role in your system.

In order to do this, we're going to get to a point where all we're maintaining is a list of specs we can update when we add new routes. 

To start, although you can do this with strings, let's put our auth Roles into an enum for typed goodness. In our application code we have: 

```kotlin
enum class Role {
    BASIC,
    ADMIN
}
```

Let's also add a class in our tests to help us understand what authorization should be granted on a route:

```kotlin
sealed class Access {
    object Unauthenticated : Access()
    class AnyRole(private vararg val roles: Role) : Access() {
        fun allowedForRole(role: Role): Boolean {
            return roles.contains(role)
        }
    }
}
```

And finally, we can bring our pieces we need to write tests together with a RouteAuthSpec:

```kotlin
data class RouteAuthSpec(
    val route: String,
    val verb: HttpMethod,
    val access: Access
) {
    val request: MockHttpServletRequestBuilder
        get() {
            val sanitizedRoute = route.replace(Regex("\\{\\w+}"), "0")
            return when (verb) {
                GET -> get(sanitizedRoute)
                POST -> post(sanitizedRoute).with(csrf())
                PUT -> put(sanitizedRoute).with(csrf())
                DELETE -> delete(sanitizedRoute).with(csrf())
                PATCH -> patch(sanitizedRoute).with(csrf())
                else -> TODO("Not Implemented for method $verb")
            }
        }

    override fun toString(): String {
        return "submits a $verb to $route"
    }
}
```

Sanitizing the route allows us to configure our RouteAuthSpecs with the exact URL we'd use in our controllers. We also don't care what params we pass as we're only looking for access, not successful response codes. 

Now we have a few of our helper classes sorted, let's bring them together into a `WebSecurityConfigTest` class and add a few routes we care about:

```kotlin
@SpringBootTest
@AutoConfigureMockMvc
internal class WebSecurityConfigTest {
    val routeAuthSpecs: Set<RouteAuthSpec> = setOf(
        RouteAuthSpec("/user/{id}", HttpMethod.GET, Access.Unauthenticated),
        RouteAuthSpec("/user/{id}", HttpMethod.PUT, Access.AnyRole(Role.ADMIN, Role.BASIC)),
    )
}
```

### Solution - Kotlin - All Routes

The first thing we want to do, is make sure we have a RouteAuthSpec for every route in our application. We can do this by using `RequestMappingHandlerMapping` to pull all registered routes out of our app and looking at differences. Here's an example of the class after adding this test:

```kotlin
@SpringBootTest
@AutoConfigureMockMvc
internal class WebSecurityConfigTest {
    val routeAuthSpecs: Set<RouteAuthSpec> = setOf(
        RouteAuthSpec("/user/{id}", HttpMethod.GET, Access.Unauthenticated),
        RouteAuthSpec("/user/{id}", HttpMethod.PUT, Access.AnyRole(Role.ADMIN, Role.BASIC)),
    )

    @Autowired
    lateinit var requestMapping: RequestMappingHandlerMapping

    @Test
    fun `tests every route and no non-existent routes`() {
        val allRoutes = requestMapping.handlerMethods.keys
            .filter { it.patternsCondition != null }
            .filterNot { it.patternsCondition!!.patterns.contains("/error") }
            .flatMap { requestMappingInfo ->
                val method = requestMappingInfo.methodsCondition.methods.first()
                val routes = requestMappingInfo.patternsCondition!!.patterns
                routes.map { route -> "$method $route" }
            }
        val testedRoutes = routeAuthSpecs.map { spec ->
            "${spec.verb} ${spec.route}"
        }

        val untestedRoutes: Set<String> = allRoutes.subtract(testedRoutes)
        val nonexistentRoutes = testedRoutes.minus("GET /index").subtract(allRoutes)

        assertAll(
            { assertTrue(untestedRoutes.isEmpty(), String.format("The following routes are untested: %s", untestedRoutes)) },
            { assertTrue(nonexistentRoutes.isEmpty(), String.format("Tests are defined for the following nonexistent routes: %s", nonexistentRoutes)) }
        )
    }
}
```
`allRoutes` is getting every method and route we have registered (minus Spring's default `/error` page). Then it's a matter of comparing it to the list of routes we have specs for.

Now, we'll have a test failure if we ever add an application route without adding a corresponding `RouteAuthSpec`. We also will get a failure if we remove a route from our app without removing the relevant `RouteAuthSpec`.

### Solution - Kotlin - Unauthenticated Routes

Next we'll want to ensure every Unauthenticated route is behaving properly:

```kotlin
    @TestFactory
    fun `all routes have correct authorization when unauthenticated`(): List<DynamicTest> =
        routeAuthSpecs.map { spec ->
            dynamicTest("Unauthenticated [${spec.verb}] ${spec.route}") {
                assertCorrectUnauthenticated(spec)
            }
        }


    private fun assertCorrectUnauthenticated(spec: RouteAuthSpec) {
        val result = mockMvc.perform(spec.request).andReturn()

        assertNotEquals(HttpStatus.METHOD_NOT_ALLOWED.value(), result.response.status,
            "Method ${spec.verb} does not exist for route ${spec.route}")

        when (spec.access) {
            is Access.Unauthenticated -> {
                assertNotEquals(HttpStatus.UNAUTHORIZED.value(), result.response.status,
                    "Expected ${spec.verb} ${spec.route} not to require authentication")
            }
            else -> {
                assertEquals(HttpStatus.UNAUTHORIZED.value(), result.response.status,
                    "Expected ${spec.verb} ${spec.route} to require authentication.")
            }
        }
    }
```

We use `@TestFacotry` here so we get an individual pass or failure for every route in our system. 

### Solution - Kotlin - Authenticated Routes

At this point, verifying authenticated routes isn't much different from unauthenticated routes:

```kotlin
    @TestFactory
    fun `all routes have correct authorization when authenticated`(): List<DynamicTest> =
        Role.values().flatMap { role ->
            routeAuthSpecs.map { spec ->
                dynamicTest("$role [${spec.verb}] ${spec.route}") {
                    assertCorrectAuthz(role, spec)
                }
            }
        }

    private fun assertCorrectAuthz(role: Role, spec: RouteAuthSpec) {
        val result = mockMvc.perform(
            spec.request.with(
                SecurityMockMvcRequestPostProcessors
                    .user("automation-user")
                    .roles(role.toString())
            )
        ).andReturn()

        when (spec.access) {
            is Access.AnyRole -> {
                if (spec.access.allowedForRole(role)) {
                    assertNotEquals(HttpStatus.FORBIDDEN.value(), result.response.status,
                        "Expected role $role to be PERMITTED to ${spec.verb} ${spec.route}")
                } else {
                    assertEquals(HttpStatus.FORBIDDEN.value(), result.response.status,
                        "Expected role $role to be FORBIDDEN to ${spec.verb} ${spec.route}")
                }
            }
            is Access.Unauthenticated -> {
                assertNotEquals(HttpStatus.FORBIDDEN.value(), result.response.status,
                    "Expected role $role to be PERMITTED to ${spec.verb} ${spec.route}")
            }
        }
    }
```


## Solution - Java

The java solution uses all the same concepts as the kotlin solution. In the end, we end up with with this test class:

```java
import org.junit.jupiter.api.DynamicTest;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.TestFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.http.HttpMethod;
import org.springframework.http.HttpStatus;
import org.springframework.security.test.web.servlet.request.SecurityMockMvcRequestPostProcessors;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.test.web.servlet.MvcResult;
import org.springframework.web.bind.annotation.RequestMethod;
import org.springframework.web.servlet.mvc.method.RequestMappingInfo;
import org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping;

import java.util.Arrays;
import java.util.List;
import java.util.Set;
import java.util.stream.Collectors;

import static org.junit.jupiter.api.Assertions.*;
import static org.junit.jupiter.api.DynamicTest.dynamicTest;

@SpringBootTest
@AutoConfigureMockMvc
class WebSecurityConfigTest {
    final Set<RouteAuthSpec> routeAuthSpecs = Set.of(
        new RouteAuthSpec("/user/{id}", HttpMethod.GET, new Access.Unauthenticated()),
        new RouteAuthSpec("/user/{id}", HttpMethod.PUT, new Access.AnyRole(Role.ADMIN, Role.BASIC)),
        new RouteAuthSpec("/user/{id}", HttpMethod.PATCH, new Access.AnyRole(Role.ADMIN, Role.BASIC)),
        new RouteAuthSpec("/user", HttpMethod.POST, new Access.AnyRole(Role.ADMIN)),
        new RouteAuthSpec("/user/{id}", HttpMethod.DELETE, new Access.AnyRole(Role.ADMIN)),
        new RouteAuthSpec("/some-route-without-correct-auth-to-show-failure", HttpMethod.POST, new Access.AnyRole(Role.ADMIN)),
        new RouteAuthSpec("/some-route-not-implemented-to-show-failure", HttpMethod.GET, new Access.Unauthenticated())
    );

    @Autowired
    MockMvc mockMvc;

    @Autowired
    RequestMappingHandlerMapping requestMapping;

    @Test
    void testsEveryRouteAndNoNonExistentRoutes() {
        Set<String> allRoutes = requestMapping.getHandlerMethods().keySet().stream()
            .filter((RequestMappingInfo info) ->
                info.getPatternsCondition() != null &&
                    !info.getPatternsCondition().getPatterns().contains("/error")
            ).flatMap((RequestMappingInfo info) -> {
                RequestMethod method = info.getMethodsCondition().getMethods().iterator().next();
                Set<String> routes = info.getPatternsCondition().getPatterns();
                return routes.stream().map((r) -> String.format("%s %s", method, r));
            }).collect(Collectors.toSet());

        Set<String> testedRoutes = routeAuthSpecs.stream()
            .map((s) -> String.format("%s %s", s.verb, s.route))
            .collect(Collectors.toSet());

        Set<String> untestedRoutes = allRoutes.stream()
            .filter((r) -> !testedRoutes.contains(r))
            .collect(Collectors.toSet());

        Set<String> nonexistentRoutes = testedRoutes.stream()
            .filter((r) -> !allRoutes.contains(r))
            .collect(Collectors.toSet());

        assertAll(
            () -> assertTrue(untestedRoutes.isEmpty(), String.format("The following routes are untested: %s", untestedRoutes)),
            () -> assertTrue(nonexistentRoutes.isEmpty(), String.format("Tests are defined for the following nonexistent routes: %s", nonexistentRoutes))
        );
    }

    @TestFactory
    List<DynamicTest> allRoutesHaveCorrectAuthorizationWhenUnauthenticated() {
        return routeAuthSpecs.stream().map((spec) -> dynamicTest(
            String.format("Unauthenticated [%s] %s", spec.verb, spec.route), () -> {
                assertCorrectUnauthenticated(spec);
            }
        )).collect(Collectors.toList());
    }

    @TestFactory
    List<DynamicTest> allRoutesHaveCorrectAuthorizationWhenAuthenticated() {
        return Arrays.stream(Role.values()).flatMap((role) ->
            routeAuthSpecs.stream().map((spec) -> dynamicTest(
                String.format("%s [%s] %s", role, spec.verb, spec.route), () -> {
                    assertCorrectAuthz(role, spec);
                }
            ))
        ).collect(Collectors.toList());
    }

    void assertCorrectUnauthenticated(RouteAuthSpec spec) throws Exception {
        MvcResult result = mockMvc.perform(spec.getRequest()).andReturn();

        assertNotEquals(HttpStatus.METHOD_NOT_ALLOWED.value(), result.getResponse().getStatus(),
            String.format("Method %s does not exist for route %s", spec.verb, spec.route));

        if (spec.access instanceof Access.Unauthenticated) {
            assertNotEquals(HttpStatus.UNAUTHORIZED.value(), result.getResponse().getStatus(),
                String.format("Expected %s %s not to require authentication", spec.verb, spec.route));
        } else {
            assertEquals(HttpStatus.UNAUTHORIZED.value(), result.getResponse().getStatus(),
                String.format("Expected %s %s to require authentication.", spec.verb, spec.route));
        }
    }


    void assertCorrectAuthz(Role role, RouteAuthSpec spec) throws Exception {
        MvcResult result = mockMvc.perform(
            spec.getRequest().with(
                SecurityMockMvcRequestPostProcessors
                    .user("automation-user")
                    .roles(role.toString())
            )
        ).andReturn();

        if (spec.access instanceof Access.AnyRole) {
            if (((Access.AnyRole) spec.access).allowedForRole(role)) {
                assertNotEquals(HttpStatus.FORBIDDEN.value(), result.getResponse().getStatus(),
                    String.format("Expected role %s to be PERMITTED to %s %s", role, spec.verb, spec.route));
            } else {
                assertEquals(HttpStatus.FORBIDDEN.value(), result.getResponse().getStatus(),
                    String.format("Expected role %s to be FORBIDDEN to %s %s", role, spec.verb, spec.route));
            }
        } else if (spec.access instanceof Access.Unauthenticated) {
            assertNotEquals(HttpStatus.FORBIDDEN.value(), result.getResponse().getStatus(),
                String.format("Expected role %s to be PERMITTED to %s %s", role, spec.verb, spec.route));
        } else {
            throw new Error("Unknown class passed for spec.access");
        }
    }
}

```

We also have these supporting classes:

```java
public class RouteAuthSpec {
    String route;
    HttpMethod verb;
    Access access;

    RouteAuthSpec(String route, HttpMethod verb, Access access) {
        this.route = route;
        this.verb = verb;
        this.access = access;
    }

    MockHttpServletRequestBuilder getRequest() {
        String sanitizedRoute = route.replaceAll("\\{\\w+}", "0");
        switch (this.verb) {
            case GET:
                return get(sanitizedRoute);
            case POST:
                return post(sanitizedRoute).with(csrf());
            case PUT:
                return put(sanitizedRoute).with(csrf());
            case PATCH:
                return patch(sanitizedRoute).with(csrf());
            case DELETE:
                return delete(sanitizedRoute).with(csrf());
            default:
                throw new Error(String.format("Method %s not implemented", this.verb.toString()));
        }
    }
}

class Access {
    static class Unauthenticated extends Access {
    }

    static class AnyRole extends Access {
        List<Role> roles;

        AnyRole(Role... roles) {
            this.roles = Arrays.stream(roles).collect(toList());
        }

        Boolean allowedForRole(Role role) {
            return this.roles.contains(role);
        }
    }
}
```

#### Pros

- Humans don't have to remember to update it, the tests tell you when to change
- Every API route is tested against every role
- `@TestFactory` generates an individual case for every route
- Easy for any team member to add new specs
- Lets you refactor your authorization matchers with confidence

#### Cons

- Adds a lot of tests to your suite, could slow it down
- Your hand will get sore from all the high fives

## Further reading
- [Dynamic tests in Junit 5](https://www.baeldung.com/junit5-dynamic-tests)
