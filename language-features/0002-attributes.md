# RFC 0002: Attributes

**Status:** Proposed

**Author:** Donald Pakkies

**Created:** 25/12/2025

## Summary

This RFC designs a comprehensive attribute system for Strata, enabling metadata annotation of classes, methods, properties, and other language constructs. Attributes provide a declarative way to attach metadata that can be accessed at runtime through reflection, enabling powerful patterns like routing, validation, dependency injection, and more.

## Motivation

**Strata currently has no way to attach metadata to code elements.** Modern applications often need this capability for various purposes:

1. **Routing**: Define HTTP routes directly on controller methods
2. **Validation**: Specify validation rules on class properties
3. **Dependency Injection**: Mark classes and methods for automatic wiring
4. **ORM Mapping**: Define database table and column mappings
5. **Authorization**: Specify access control rules
6. **Serialization**: Control how objects are serialized/deserialized
7. **Testing**: Mark test methods and configure test behavior

Without a metadata system, the only options are workarounds:
- **External configuration files**: Disconnected from code, hard to maintain
- **Manual registration**: Verbose, error-prone, boilerplate-heavy
- **Naming conventions**: Implicit, not enforced, easy to break

**This RFC proposes adding attributes** to provide a first-class, type-safe way to attach metadata directly to code elements, enabling declarative programming patterns that are common in modern frameworks.

## Design Goals

1. **Type Safety**: Attributes are classes with full type checking
2. **Reflection Access**: Metadata accessible at runtime via reflection
3. **PHP Compatibility**: Compile cleanly to PHP 8+ attributes
4. **Declarative Syntax**: Clean, readable syntax using `@` prefix
5. **Target Validation**: Ensure attributes are only used on valid targets
6. **Composability**: Multiple attributes can be applied to the same element
7. **Inheritance**: Attributes can be inherited from parent classes

## Design

### 1. Attribute Declaration

Attributes are declared using the `attribute` keyword:

```strata
attribute Route(path: String, method: String = "GET") targets Method {
    // Optional: attach method for custom logic
    fn attach(methodInfo: ReflectionMethod): Void {
        // Called when attribute is instantiated
        // Has access to the attributed method via methodInfo parameter
    }
}
```

**Key Components:**

- **`attribute` keyword**: Declares an attribute class
- **Parameters**: Constructor parameters (with optional defaults)
- **`targets` clause**: Specifies where the attribute can be used
- **`attach()` method**: Optional lifecycle hook

### 2. Attribute Targets

The `targets` clause specifies valid application points:

```strata
attribute Route(...) targets Method { }                // Only on methods
attribute Column(...) targets Property { }             // Only on properties
attribute Table(...) targets Class { }                 // Only on classes
attribute Deprecated(...) targets Method, Property { } // Multiple targets
attribute Injectable(...) targets Class, Method { }    // Multiple targets
```

**Available Targets:**

- `Class`: Class declarations
- `Method`: Method declarations
- `Property`: Property declarations
- `Parameter`: Function/method parameters
- `Function`: Top-level functions
- `Constant`: Constants

**Combining Targets:**

Use `,` to allow multiple targets:

```strata
attribute Cache(ttl: Int) targets Method, Function {
    // Can be used on both methods and functions
}
```

### 3. Applying Attributes

Attributes are applied using the `@` prefix:

```strata
@Route(path: "/users/{id}", method: "GET")
public fn getUser(id: Int): User {
    // Method implementation
}
```

**Multiple Attributes:**

```strata
@Route(path: "/admin/users", method: "POST")
@RequiresAuth(role: "admin")
@RateLimit(requests: 10, window: 60)
public fn createUser(data: UserData): User {
    // Method implementation
}
```

**Attributes on Classes:**

```strata
@Table(name: "users")
@Cache(ttl: 3600)
class User(id: Int, name: String, email: String) {
    @Column(name: "user_id", primary: true)
    private id: Int;

    @Column(name: "full_name")
    private name: String;

    @Column(name: "email_address", unique: true)
    private email: String;
}
```

**Attributes on Properties:**

```strata
class UserForm {
    @Required
    @MinLength(value: 3)
    @MaxLength(value: 50)
    public name: String;

    @Required
    @Email
    public email: String;

    @Min(value: 18)
    @Max(value: 120)
    public age: Int;
}
```

### 4. Attribute Parameters

Attributes only support named parameters:

```strata
// Named parameters (required)
@Route(path: "/users", method: "GET")

// Default values
attribute Route(path: String, method: String = "GET") targets Method { }

@Route(path: "/users")  // method defaults to "GET"

// Positional parameters are NOT supported
@Route("/users", "GET")  // ❌ Error: must use named parameters
```

**Rationale:** Named parameters make attribute usage more explicit and readable, especially when attributes have multiple parameters or optional parameters.

### 5. Accessing Attributes via Reflection

Attributes are accessed at runtime using reflection:

```strata
import Reflection;

class UserController {
    @Route(path: "/users/{id}", method: "GET")
    public fn getUser(id: Int): User {
        // ...
    }
}

// Get method reflection
let reflector = Reflection.getMethod(UserController, "getUser");

// Get all attributes
let attributes = reflector.getAttributes();

// Get specific attribute type
let routeAttributes = reflector.getAttributes(Route);

// Iterate and instantiate
for attribute in routeAttributes {
    let route = attribute.newInstance();
    echo "Route: " + route.method + " " + route.path;
    // Output: Route: GET /users/{id}
}
```

### 6. Attribute Lifecycle: `attach()` Method

The optional `attach()` method enables **automatic reflection-based execution**. When an attribute defines an `attach()` method, the Strata runtime automatically:

1. Scans for all uses of that attribute
2. Instantiates the attribute with its parameters
3. Calls the `attach()` method with context about the attributed element

This eliminates the need for manual reflection in your application code.

#### `attach()` Method Signature

The `attach()` method **must accept a parameter** that provides context about what it's attached to:

- **For `Method` targets**: Receives method reflection information
- **For `Property` targets**: Receives the property value
- **For `Class` targets**: Receives an instance of the class

**Example: Method Attribute with Reflection Context**

```strata
attribute Route(path: String, method: String = "GET") targets Method {
    fn attach(methodInfo: ReflectionMethod): Void {
        // Access method information
        let className = methodInfo.getDeclaringClass().getName();
        let methodName = methodInfo.getName();

        // Register route with full context
        Router.register(this.method, this.path, className, methodName);
    }
}

class UserController {
    @Route(path: "/users/{id}", method: "GET")
    public fn getUser(id: Int): User {
        // Route is automatically registered at startup
    }
}
```

**Example: Property Attribute with Value Context**

```strata
attribute Validate(rules: Array<String>) targets Property {
    fn attach(value: Mixed): Void {
        // Access property value
        // Register validation rules with the value
        Validator.register(this.rules, value);
    }
}

class UserForm {
    @Validate(rules: ["required", "email"])
    public email: String = "";
}
```

**Example: Class Attribute with Instance Context**

```strata
attribute Injectable() targets Class {
    fn attach(instance: Mixed): Void {
        // Access class instance
        let className = typeof(instance);

        // Register with dependency injection container
        Container.register(className, instance);
    }
}

@Injectable
class UserService {
    // Automatically registered with DI container
}
```

**Without `attach()` - Manual Reflection Required:**

```strata
attribute Cache(ttl: Int) targets Method {
    // No attach() method
}

// You must manually use reflection
let reflector = Reflection.getMethod(UserController, "getUser");
let attributes = reflector.getAttributes(Cache);
for attr in attributes {
    let cache = attr.newInstance();
    // Manually process cache attribute
}
```

**When to Use `attach()`:**

- ✅ **Use `attach()`** when you want automatic registration/processing (routes, event listeners, validators)
- ❌ **Don't use `attach()`** when you need lazy/on-demand processing (caching metadata, serialization hints)

**Execution Timing:**

The `attach()` method is called during application initialization, after all classes are loaded but before the main application logic runs.

### 7. PHP Attribute Interoperability

Strata can use existing PHP attributes directly, enabling seamless integration with PHP libraries and frameworks:

```strata
// Using PHP's built-in attributes
import Attribute;
import Override;

class User {
    @Override
    public fn toString(): String {
        return this.name;
    }
}
```

**Using Third-Party PHP Attributes:**

```strata
// Symfony routing
import Symfony.Component.Routing.Annotation.Route as SymfonyRoute;

class UserController {
    @SymfonyRoute(path: "/users/{id}", methods: ["GET"])
    public fn getUser(id: Int): User {
        // ...
    }
}

// Doctrine ORM
import Doctrine.ORM.Mapping.Entity;
import Doctrine.ORM.Mapping.Table;
import Doctrine.ORM.Mapping.Column;

@Entity
@Table(name: "users")
class User(id: Int, name: String) {
    @Column(name: "user_id", type: "integer")
    private id: Int;

    @Column(name: "full_name", type: "string")
    private name: String;
}
```

**Mixing Strata and PHP Attributes:**

```strata
// Define Strata attribute
attribute Cache(ttl: Int) targets Method { }

// Use both Strata and PHP attributes
import Symfony.Component.Routing.Annotation.Route as SymfonyRoute;

class UserController {
    @SymfonyRoute(path: "/users", methods: ["GET"])
    @Cache(ttl: 3600)
    public fn listUsers(): Array<User> {
        // ...
    }
}
```

**Reflection Works for Both:**

```strata
// Get all attributes (Strata and PHP)
let reflector = Reflection.getMethod(UserController, "listUsers");
let allAttributes = reflector.getAttributes();

// Get specific PHP attribute
let routeAttributes = reflector.getAttributes(SymfonyRoute);

// Get specific Strata attribute
let cacheAttributes = reflector.getAttributes(Cache);
```

**Rationale:** This allows Strata to leverage the entire PHP ecosystem without requiring wrapper attributes or special handling.

### 8. Built-in Attributes

Strata can provide built-in attributes:

```strata
// Deprecation
@Deprecated(message: "Use newMethod() instead", since: "2.0")
public fn oldMethod(): Void { }

// Override marker
@Override
public fn toString(): String { }

// Serialization control
@JsonIgnore
private password: String;

@JsonProperty(name: "user_email")
private email: String;
```

### 9. Compilation to PHP

Strata attributes compile directly to PHP 8+ `#[Attribute]` syntax.

**Attribute Declaration:**

```strata
attribute Route(path: String, method: String = "GET") targets Method {
    fn attach(methodInfo: ReflectionMethod): Void {
        Router.register(this.method, this.path, methodInfo);
    }
}
```

Compiles to:

```php
<?php
#[Attribute(Attribute::TARGET_METHOD)]
class Route {
    public function __construct(
        public string $path,
        public string $method = 'GET'
    ) {}

    public function attach($methodInfo): void {
        Router::register($this->method, $this->path, $methodInfo);
    }
}
```

**Attribute Usage:**

```strata
@Route(path: "/users/{id}", method: "GET")
public fn getUser(id: Int): User { }
```

Compiles to:

```php
#[Route(path: '/users/{id}', method: 'GET')]
public function getUser(int $id): User { }
```

**Automatic `attach()` Execution:**

The Strata compiler injects attribute processing code at the top of the entry file:

```php
<?php
// === AUTO-GENERATED (added by Strata compiler) ===
foreach ([UserController::class, ...] as $className) {
    $reflection = new ReflectionClass($className);
    foreach ($reflection->getMethods() as $method) {
        foreach ($method->getAttributes() as $attr) {
            $instance = $attr->newInstance();
            if (method_exists($instance, 'attach')) {
                $instance->attach($method);
            }
        }
    }
    // Similar loops for properties and class attributes
}
// === END AUTO-GENERATED ===

function main() { /* your code */ }
main();
```

**Target Mapping:**

| Strata Target | PHP Target |
|---------------|------------|
| `Method` | `Attribute::TARGET_METHOD` |
| `Property` | `Attribute::TARGET_PROPERTY` |
| `Class` | `Attribute::TARGET_CLASS` |
| `Parameter` | `Attribute::TARGET_PARAMETER` |
| `Function` | `Attribute::TARGET_FUNCTION` |
| `Constant` | `Attribute::TARGET_CLASS_CONSTANT` |

Multiple targets use bitwise OR: `Attribute::TARGET_METHOD \| Attribute::TARGET_PROPERTY`

If an attribute doesn't have `attach()`:

```strata
attribute Cache(ttl: Int) targets Method { }
```

You must manually use reflection to access its values:

```php
<?php
// Manual reflection required
$reflector = new ReflectionMethod(UserController::class, 'getUser');
$attributes = $reflector->getAttributes(Cache::class);

foreach ($attributes as $attr) {
    $cache = $attr->newInstance();
    // Process cache manually
}
```

#### 10.4 Attribute Usage

```strata
// Strata
class UserController {
    @Route(path: "/users/{id}", method: "GET")
    public fn getUser(id: Int): User {
        // ...
    }
}
```

Compiles to:

```php
<?php
class UserController {
    #[Route(path: '/users/{id}', method: 'GET')]
    public function getUser(int $id): User {
        // ...
    }
}
```

#### 10.5 Target Mapping

Strata targets map to PHP attribute targets:

| Strata Target | PHP Target |
|---------------|------------|
| `Class` | `Attribute::TARGET_CLASS` |
| `Method` | `Attribute::TARGET_METHOD` |
| `Property` | `Attribute::TARGET_PROPERTY` |
| `Parameter` | `Attribute::TARGET_PARAMETER` |
| `Function` | `Attribute::TARGET_FUNCTION` |
| `Constant` | `Attribute::TARGET_CLASS_CONSTANT` |

Multiple targets use comma separation:

```strata
// Strata
attribute Cache(...) targets Method, Function { }
```

Compiles to:

```php
<?php
#[Attribute(Attribute::TARGET_METHOD | Attribute::TARGET_FUNCTION)]
class Cache { }
```

## Grammar Changes

### Updated EBNF

```ebnf
// Attribute declaration
attribute_decl
    ::= annotations "attribute" IDENTIFIER
        "(" parameter_list? ")"
        "targets" target_list
        block
    ;

target_list
    ::= target ("," target)*
    ;

target
    ::= "Class"
     | "Method"
     | "Property"
     | "Parameter"
     | "Function"
     | "Constant"
    ;

// Attribute usage (annotations)
annotations
    ::= annotation*
    ;

annotation
    ::= "@" IDENTIFIER "(" argument_list? ")"
    ;

argument_list
    ::= argument ("," argument)*
    ;

argument
    ::= (IDENTIFIER ":")? expression
    ;
```

## Type Checking Rules

### Attribute Declaration Rules

1. **Parameters**: Attribute parameters must have types
2. **Targets**: Must specify at least one valid target
3. **attach() Method**: If present, must return `Void` and accept exactly one parameter:
   - For `Method` targets: `ReflectionMethod` parameter
   - For `Property` targets: `Mixed` parameter (the property value)
   - For `Class` targets`: `Mixed` parameter (the class instance)
4. **Properties**: Attributes can have properties (from constructor parameters)

### Attribute Usage Rules

1. **Target Validation**: Attribute can only be used on declared targets
2. **Parameter Matching**: Arguments must match attribute constructor signature
3. **Type Checking**: Argument types must match parameter types
4. **Multiple Attributes**: Same attribute can be applied multiple times if allowed
5. **Required Parameters**: Non-optional parameters must be provided

### Reflection Rules

1. **getAttributes()**: Returns array of attribute instances
2. **getAttributes(Type)**: Returns array of specific attribute type
3. **newInstance()**: Instantiates attribute with provided arguments

## Implementation Plan

### Phase 1: Parser
- Add `attribute_decl` to grammar
- Parse `attribute` keyword and `targets` clause
- Parse `@` annotations on declarations
- Validate syntax rules

### Phase 2: AST
- Add `AttributeDecl` AST node
- Add `Annotation` AST node
- Update declaration nodes to include annotations
- Store target information

### Phase 3: Type Checking
- Validate attribute declarations
- Check target usage matches declaration
- Validate attribute arguments match constructor
- Type check attribute parameters

### Phase 4: Code Generation
- Emit PHP `#[Attribute]` for attribute declarations
- Emit PHP attribute targets (bitwise OR for multiple)
- Emit PHP `#[AttributeName]` for usage
- Generate constructor with property promotion

### Phase 5: Reflection API
- Design reflection API for accessing attributes
- Implement `getAttributes()` methods
- Implement `newInstance()` for attribute instantiation

### Phase 6: Testing
- Unit tests for parsing
- Type checking tests
- Code generation tests
- Integration tests with reflection

## Edge Cases

### 1. Attribute on Attribute

Can attributes be applied to attribute declarations?

```strata
@Deprecated
attribute OldRoute(...) targets Method { }
```

**Recommendation:** Allow it for metadata purposes.

### 2. Repeatable Attributes

Can the same attribute be applied multiple times?

```strata
@Route(path: "/users", method: "GET")
@Route(path: "/api/users", method: "GET")
public fn getUsers(): Array<User> { }
```

**Recommendation:** Allow by default (matches PHP 8 behavior).

### 3. Attribute Inheritance

Which attributes are inherited?

```strata
@Cache(ttl: 3600)
class BaseController { }

class UserController : BaseController {
    // Does it inherit @Cache?
}
```

**Recommendation:** Make inheritance opt-in via attribute flag or separate design.

### 4. Attribute Validation

When are attribute targets validated?

```strata
@Route(path: "/users")  // Used on method (valid)
class User { }          // Used on class (invalid - compile error)
```

**Recommendation:** Validate at compile time.

### 5. Multiple Targets with `attach()`

What happens when an attribute targets multiple element types and has an `attach()` method?

```strata
attribute Cache(ttl: Int) targets Method, Property {
    fn attach(context: ???): Void {
        // What type should context be?
        // ReflectionMethod for methods?
        // Mixed for properties?
    }
}
```

**Options:**

**Option A: Disallow `attach()` for multi-target attributes**
```strata
attribute Cache(ttl: Int) targets Method, Property {
    // ❌ Error: attach() not allowed when targeting multiple types
    fn attach(context: Mixed): Void { }
}
```

**Option B: Use union type or generic `Mixed`**
```strata
attribute Cache(ttl: Int) targets Method, Property {
    fn attach(context: ReflectionMethod | Mixed): Void {
        // Runtime type checking required
        if context is ReflectionMethod {
            // Handle method
        } else {
            // Handle property value
        }
    }
}
```

**Option C: Separate attach methods per target**
```strata
attribute Cache(ttl: Int) targets Method, Property {
    fn attachMethod(methodInfo: ReflectionMethod): Void { }
    fn attachProperty(value: Mixed): Void { }
}
```

**Recommendation:** Option A (disallow `attach()` for multi-target attributes) is simplest and clearest. Multi-target attributes should use manual reflection.

## Open Questions

1. **Repeatability**: Should attributes be repeatable by default?
2. **Reflection API**: What should the full reflection API look like?
3. **Built-in Attributes**: Which built-in attributes should Strata provide?
4. **Attribute Composition**: Can attributes reference other attributes?
5. **Runtime vs Compile-time**: Should some attributes be processed at compile time?
6. **`attach()` Execution Order**: What order should `attach()` methods be called when multiple attributes are present?
7. **`attach()` Error Handling**: What happens if an `attach()` method throws an error during initialization?
8. **`attach()` with Multiple Targets**: Should `attach()` be allowed for attributes that target multiple element types? If so, what should the parameter type be?

## Alternatives Considered

1. **Docblock Annotations**: Use PHP-style docblock comments
   - **Rejected**: Not type-safe, fragile parsing, no IDE support

2. **Decorator Pattern**: Use wrapper functions/classes
   - **Rejected**: Verbose, runtime overhead, less declarative

3. **Configuration Files**: External metadata files
   - **Rejected**: Disconnected from code, harder to maintain

4. **Prefix with `#` instead of `@`**: Match PHP syntax exactly
   - **Rejected**: `@` is more common in other languages and clearer

5. **No `targets` clause**: Allow attributes anywhere
   - **Rejected**: Less safe, harder to catch errors

6. **Positional Parameters**: Allow positional parameters like `@Route("/path", "GET")`
   - **Rejected**: Named parameters are more explicit and readable, especially with optional parameters

## References

- [PHP Attributes](https://www.php.net/manual/en/language.attributes.php)
- [Java Annotations](https://docs.oracle.com/javase/tutorial/java/annotations/)
- [C# Attributes](https://learn.microsoft.com/en-us/dotnet/csharp/programming-guide/concepts/attributes/)
- [Python Decorators](https://docs.python.org/3/glossary.html#term-decorator)
