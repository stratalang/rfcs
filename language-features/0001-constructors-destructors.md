# RFC 0001: Constructors and Destructors

**Status:** Proposed

**Author:** Donald Pakkies

**Created:** 20/12/2025

## Summary

This RFC designs explicit constructors and destructors for Strata classes, providing fine-grained control over object initialization and cleanup while maintaining compatibility with the existing property promotion system.

## Motivation

Currently, Strata classes use property promotion where constructor parameters automatically become properties. While this is concise, there are cases where we need:

1. **Custom initialization logic**: Validation, computed properties, resource acquisition
2. **Explicit control**: When property promotion isn't sufficient
3. **Cleanup logic**: Resource deallocation, connection closing, etc.
4. **Integration with existing code**: PHP classes often have complex constructors

This RFC provides a complete design for both constructors and destructors.

## Design Goals

1. **Compatibility**: Work seamlessly with property promotion
2. **Explicitness**: Clear syntax that's easy to understand
3. **Type Safety**: Full type checking for constructor/destructor parameters
4. **PHP Compatibility**: Compile cleanly to PHP's `__construct` and `__destruct`
5. **Inheritance**: Proper handling of parent constructors/destructors
6. **Safety**: Prevent common pitfalls (e.g., accessing uninitialized properties)

## Design

### 1. Constructor Syntax

#### 1.1 Explicit Constructor Declaration

Constructors are declared using the `fn init` keyword (or alternative: `fn constructor`):

```strata
class User(name: String, email: String) {
    // Property promotion still works
    // name and email are automatically properties

    // Explicit constructor for additional initialization
    public fn init(): Void {
        // Custom initialization logic - validate email format
        if strpos(this.email, "@") === false {
            throw Error("Invalid email format");
        }
    }
}
```

**Alternative Syntax Options:**

**Option A: `fn init` (recommended)**
- Short and clear
- Differentiates from regular methods
- Consistent naming pattern

**Option B: `fn constructor`**
- More explicit about what it is
- Longer but self-documenting

**Option C: `fn __construct`**
- Matches PHP directly
- Less "Strata-like"

**Recommendation:** Use `fn init` for clean, concise syntax that fits Strata's design philosophy.

#### 1.2 Constructor with Parameters

If a class needs constructor parameters beyond property promotion:

```strata
class DatabaseConnection {
    private connection: Resource?;

    public fn init(host: String, port: Int, database: String): Void {
        // Initialize connection
        this.connection = connect(host: host, port: port, database: database);
    }
}
```

**Question:** Should we allow constructor parameters when property promotion is also used?

```strata
// Option 1: Mix property promotion + constructor params
class User(name: String, email: String) {
    private id: String;

    public fn init(id: String): Void {
        this.id = id;
    }
}

// Option 2: Only one or the other
// If you need constructor params, don't use property promotion
```

**Recommendation:** Allow mixing, but constructor params don't become properties automatically (only class header params do).

#### 1.3 Constructor Return Type

Constructors must return `Void`:

```strata
public fn init(): Void {
    // ...
}
```

**Rationale:** Constructors don't return values, they initialize objects.

### 2. Destructor Syntax

#### 2.1 Basic Destructor

Destructors are declared using `fn destroy` (or alternative: `fn destructor`):

```strata
class DatabaseConnection {
    private connection: Resource?;

    public fn init(host: String, port: Int): Void {
        this.connection = connect(host: host, port: port);
    }

    public fn destroy(): Void {
        // Cleanup logic
        if this.connection !== null {
            disconnect(this.connection);
        }
    }
}
```

**Alternative Syntax Options:**

**Option A: `fn destroy` (recommended)**
- Matches `init` naming pattern
- Clear and concise

**Option B: `fn destructor`**
- More explicit
- Matches constructor naming

**Option C: `fn __destruct`**
- Matches PHP directly
- Less "Strata-like"

**Recommendation:** Use `fn destroy` to match `init` pattern.

#### 2.2 Destructor Return Type

Destructors must return `Void` and take no parameters:

```strata
public fn destroy(): Void {
    // Cleanup logic
}
```

**Rationale:** PHP destructors don't take parameters and don't return values.

### 3. Interaction with Property Promotion

#### 3.1 Execution Order

When both property promotion and explicit constructor exist:

1. **Property promotion happens first** (implicit)
2. **Explicit constructor runs second** (explicit `init()`)

```strata
class User(name: String, email: String) {
    private id: String;

    public fn init(): Void {
        // At this point, this.name and this.email are already set
        this.id = generateId();
    }
}
```

#### 3.2 Validation in Constructor

Constructors can validate promoted properties:

```strata
class User(name: String, email: String) {
    public fn init(): Void {
        // Validate email format (name is required, so always set)
        if strpos(this.email, "@") === false {
            throw Error("Invalid email format");
        }

        // Validate minimum name length
        if strlen(this.name) < 2 {
            throw Error("Name must be at least 2 characters");
        }
    }
}
```

### 4. Inheritance

#### 4.1 Parent Constructor Calls

When a class inherits from another class, parent constructors must be called explicitly:

```strata
class Person(name: String, age: Int) {
    public fn init(): Void {
        // Person initialization
    }
}

class User(name: String, age: Int, email: String) : Person(name: name, age: age) {
    public fn init(): Void {
        // Must call parent constructor
        super.init();

        // User-specific initialization - validate email format
        if strpos(this.email, "@") === false {
            throw Error("Invalid email format");
        }
    }
}
```

**Syntax for parent constructor call:**

**Option A: `super.init()`**
- Matches method call syntax
- Clear intent

**Option B: `super()`**
- Shorter
- Common in other languages

**Option C: `parent::init()`**
- Matches PHP's `parent::__construct()`
- More verbose

**Recommendation:** Use `super.init()` for consistency with method calls.

#### 4.2 Parent Destructor Calls

Parent destructors are called automatically after child destructor:

```strata
class BaseConnection {
    public fn destroy(): Void {
        // Base cleanup
    }
}

class DatabaseConnection : BaseConnection {
    public fn destroy(): Void {
        // Child cleanup first
        disconnect();

        // Parent cleanup happens automatically after
    }
}
```

**Question:** Should we require explicit `super.destroy()` call, or make it automatic?

**Recommendation:** Make it automatic (matches PHP behavior), but allow explicit `super.destroy()` if needed.

### 5. Visibility

#### 5.1 Constructor Visibility

Constructors should be `public` (default) or `protected`:

```strata
class User(name: String) {
    public fn init(): Void { }      // ✅ Public (default)
    protected fn init(): Void { }   // ✅ Protected (for base classes)
    private fn init(): Void { }     // ❌ Error: Constructors cannot be private
}
```

**Rationale:**
- Public: Normal classes (default)
- Protected: Base classes (subclasses call it)
- Private: Not allowed (can't instantiate)

#### 5.2 Destructor Visibility

Destructors should always be `public`:

```strata
class User {
    public fn destroy(): Void { }   // ✅ Public (required)
    protected fn destroy(): Void { } // ❌ Error: Destructors must be public
    private fn destroy(): Void { }   // ❌ Error: Destructors must be public
}
```

**Rationale:** PHP destructors are always public and called automatically.

### 6. Multiple Constructors / Overloading

**Question:** Should we support constructor overloading?

```strata
// Option 1: Multiple constructors (not recommended)
class User {
    public fn init(name: String): Void { }
    public fn init(name: String, email: String): Void { }
}

// Option 2: Single constructor with optional/default params (recommended)
class User {
    public fn init(name: String, email: String? = null): Void {
        this.name = name;
        if email != null {
            this.email = email;
        } else {
            this.email = "";
        }
    }
}
```

**Recommendation:** No overloading. Use default parameters instead (matches Strata's function design).

### 7. Static Constructors / Factory Methods

For alternative construction patterns, use static factory methods:

```strata
class User(name: String, email: String) {
    public static fn fromEmail(email: String): User {
        // Parse email to extract name, etc.
        let name = extractName(email: email);
        return User(name: name, email: email);
    }

    public static fn anonymous(): User {
        return User(name: "Anonymous", email: "anonymous@example.com");
    }
}
```

### 8. Error Handling

Constructors can throw errors:

```strata
class User(name: String, email: String) {
    public fn init(): Void {
        // Validate email format
        if strpos(this.email, "@") === false {
            throw Error("Invalid email format");
        }
    }
}
```

If constructor throws, object is not created (matches PHP behavior).

### 9. Compilation to PHP

#### 9.1 Constructor Compilation

```strata
// Strata
class User(name: String, email: String) {
    public fn init(): Void {
        // validation
    }
}
```

Compiles to:

```php
<?php
class User {
    public function __construct(
        private string $name,
        private string $email
    ) {
        // validation code from init()
    }
}
```

#### 9.2 Destructor Compilation

```strata
// Strata
class User {
    public fn destroy(): Void {
        // cleanup
    }
}
```

Compiles to:

```php
<?php
class User {
    public function __destruct() {
        // cleanup code from destroy()
    }
}
```

#### 9.3 Inheritance Compilation

```strata
// Strata
class User(name: String) : Person(name: name) {
    public fn init(): Void {
        super.init();
        // user init
    }
}
```

Compiles to:

```php
<?php
class User extends Person {
    public function __construct(string $name) {
        parent::__construct($name);
        // user init code
    }
}
```

## Grammar Changes

### Updated EBNF

```ebnf
class_decl
    ::= annotations "readonly"? "class" IDENTIFIER
        "(" parameter_list? ")"
        inheritance?
        trait_use?
        interface_use?
        block
    ;

class_member
    ::= function_decl
     | property_decl
     | constructor_decl
     | destructor_decl
    ;

constructor_decl
    ::= annotations visibility? "fn" "init"
        "(" parameter_list? ")"
        ":" "Void"
        block
    ;

destructor_decl
    ::= annotations "public" "fn" "destroy"
        "(" ")"
        ":" "Void"
        block
    ;
```

## Type Checking Rules

### Constructor Rules

1. **Return Type**: Must be `Void`
2. **Visibility**: Must be `public` or `protected` (not `private`)
3. **Parameters**: Can have parameters (but they don't become properties unless in class header)
4. **Property Access**: Can access promoted properties (they're initialized first)
5. **Super Calls**: Must call `super.init()` if parent has constructor (unless parent has no constructor)
6. **Throwing**: Can throw errors

### Destructor Rules

1. **Return Type**: Must be `Void`
2. **Parameters**: Must have no parameters
3. **Visibility**: Must be `public`
4. **Super Calls**: Parent destructor called automatically (optional explicit call allowed)
5. **Throwing**: Should not throw (PHP destructors that throw cause issues)

## Edge Cases

### 1. Class with Only Property Promotion

```strata
class User(name: String) {
    // No explicit constructor - property promotion handles everything
}
```

**Behavior:** Compiles to PHP constructor with property promotion only.

### 2. Class with Only Explicit Constructor

```strata
class DatabaseConnection {
    private connection: Resource?;

    public fn init(host: String): Void {
        this.connection = connect(host: host);
    }
}
```

**Behavior:** Compiles to PHP constructor with explicit parameters.

### 3. Class with Both

```strata
class User(name: String, email: String) {
    private id: String;

    public fn init(id: String): Void {
        this.id = id;
    }
}
```

**Behavior:** Class header params become properties, constructor params are just parameters.

### 4. Protected Constructors in Base Classes

```strata
class Base {
    protected fn init(): Void {
        // base init
    }
}

class Derived : Base {
    public fn init(): Void {
        super.init();
        // derived init
    }
}
```

**Behavior:** Protected constructor in base class, public in derived class.

### 5. Readonly Classes

```strata
readonly class User(name: String) {
    public fn init(): Void {
        // Can validate, but cannot modify properties after
    }
}
```

**Behavior:** Constructor can validate, but properties are readonly after construction.

## Open Questions

1. **Constructor Syntax**: `fn init` vs `fn constructor` vs `fn __construct`?
2. **Destructor Syntax**: `fn destroy` vs `fn destructor` vs `fn __destruct`?
3. **Parent Call Syntax**: `super.init()` vs `super()` vs `parent::init()`?
4. **Mixing Property Promotion + Constructor Params**: Allow or restrict?
5. **Destructor Super Calls**: Automatic or explicit?
6. **Constructor Overloading**: Support or use default params only?
7. **Private Constructors**: Allow for singleton pattern?
8. **Static Constructors**: Support or use factory methods only?

## Alternatives Considered

1. **No explicit constructors**: Keep only property promotion
   - **Rejected**: Too limiting for complex initialization

2. **Only `init()` method**: No separate constructor
   - **Rejected**: Need distinction between construction and initialization

3. **PHP-style `__construct`**: Match PHP exactly
   - **Rejected**: Prefer Strata's cleaner syntax

4. **Automatic parent calls**: Always call parent constructor/destructor
   - **Partially accepted**: Automatic for destructors, explicit for constructors

## Implementation Plan

### Phase 1: Parser
- Add `constructor_decl` and `destructor_decl` to grammar
- Parse `fn init` and `fn destroy` syntax
- Validate syntax rules (return type, parameters, visibility)

### Phase 2: AST
- Add `ConstructorDecl` and `DestructorDecl` AST nodes
- Update `ClassDecl` to include constructors/destructors

### Phase 3: Type Checking
- Validate constructor/destructor signatures
- Check super calls in inheritance
- Validate property access rules
- Check visibility rules

### Phase 4: Code Generation
- Emit PHP `__construct` from constructors
- Emit PHP `__destruct` from destructors
- Handle parent calls correctly
- Merge property promotion with explicit constructors

### Phase 5: Testing
- Unit tests for parsing
- Type checking tests
- Code generation tests
- Integration tests with inheritance

## References

- [PHP Constructors](https://www.php.net/manual/en/language.oop5.decon.php)
- [PHP Destructors](https://www.php.net/manual/en/language.oop5.decon.php#language.oop5.decon.destructor)

