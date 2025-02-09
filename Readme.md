## Install

```
composer require joe.szeto/capsule
```

## Basic usage

we can use attribute ``` Setter ``` to tell the code
the following Closure is a setter, and the value of the setter is the return value of the closure

```php
$me = capsule(
    #[Setter('name')]
    fn() => 'szeto',
    #[Setter('age')]
    fn() => 30,
    #[Setter('sex')]
    fn() => 'man'
)->thenReturn(fn(string $name, int $age, string $sex) => [ $name, $age, $sex ]);
// to be ['szeto', 30, 'man']
```

we can also use method ``` set ``` to set the value of the setter

```php
$me = capsule()
    ->set('name', fn() => 'szeto')
    ->set('age', fn() => 30)
    ->set('sex', fn() => 'man')
    ->thenReturn(fn(string $name, int $age, string $sex) => [ $name, $age, $sex ]);
// to be ['szeto', 30, 'man']
```

## Auto resolve params

if the closure has type hints and the type hint is the same as the value of the setter, the value will be resolved
automatically
even if the params name is not 100% match, it will still work

```php
$name = capsule()
    ->set('name', 'szeto')
    ->thenReturn(fn(string $myName) => $myName);
```

we can resolve parameters by the application container
leverages the underlying container's reflection capabilities. This means that when a closure or callable object is
executed, the Capsule library can automatically inject the required dependencies as defined by the type hints of the
parameters.

```php
capsule()
    ->thenReturn(fn(WhenEmpty $empty) => expect($empty)->toBeInstanceOf(WhenEmpty::class));
```

## OnBlank

it will only call when the value of the setter is null

```php
$name = capsule()
    ->set('name', fn() => null)
    ->through(
        #[OnBlank('name'), Setter('name')]
        fn() => 'szeto' // this closure only call when the value of 'name' is null
    )->thenReturn('name');
```

we can also use attribute ``` SetOnBlank ``` to set and detect the OnBlank at the same time

```php
 $name = capsule()
        ->set('name', fn() => null)
        ->through(
            #[SetOnBlank('name')]
            fn() => 'szeto'
        )
        ->thenReturn('name');
```

we can also use method ``` setOnBlank ``` to set the OnBlank

```php
$name = capsule()
    ->set('name', fn() => null)
    ->setOnBlank('name', fn() => 'szeto')
    ->thenReturn('name');
```

### Closure

if the set value is a closure,
when the type hint of the param of using is Closure
the original closure will be passed to the param

```php
    $name = capsule()
        ->set('name', fn() => 'szeto')
        ->thenReturn(fn(Closure $name) => $name());
    expect($name)->toBe('szeto');
```

when the type hint of the params is a NOT closure,
the value will be return

```php
    $name = capsule()
        ->set('name', fn() => 'szeto')
        ->thenReturn(
            fn(string $name) => $name // now name is szeto, not a closure
        );
    expect($name)->toBe('szeto');
```

### Evaluable

if the type is closure, and call it we have to pass the params manually

```php
    $name = capsule()
        ->set('prefix', fn() => 'Joe')
        ->set('name', fn(string $prefix) => $prefix. ' szeto')
        ->thenReturn(
            fn(Closure $name, string $prefix) => $name($prefix) // params is Joe
        );
```

but if we pass the param one by one manually it will be very tedious
now Evaluable become handy

```php
capsule()
    ->set('prefix', fn() => 'Joe')
    ->set('name', fn(string $prefix) => $prefix. ' szeto')
    ->thenReturn(
        fn(Evaluable $name) => $name() // params is Joe
    );
```

### Each

for each value, the closure will be called

```php
$names = capsule()
    ->set('names', fn() => ['szeto', 'joe'])
    ->through(
        #[Each('names' as: 'name')]
        fn(string $name) => $name // 'szeto', 'joe'
    )->run();
```

### Only

```php
capsule()
    ->through(
        fn() => throw new Exception('should not run'),
        #[Only]
        fn() => expect(true)->toBeTrue()
    )
    ->run();
```

### Skip

```php
capsule()
    ->through(
        fn() => expect(true)->toBeTrue(),
        #[Skip]
        fn() => throw new Exception('should not run'),
    )
    ->run();
```

### Mocking

The Capsule::mock method allows developers to replace parts of their application's behavior with predetermined responses
or operations.
This is particularly useful in testing, where you want to isolate the part of the application you are testing and
control its interactions with external dependencies.

```php
public static function mock(string $key, mixed $value): void
```

To replace a string value within the capsule, simply pass the key and the new string value to the mock method.

```php
Capsule::mock('name', 'szeto');
```

If you need to mock the behavior of a function, provide a closure as the second argument. This closure will be executed
in place of the original function associated with the given key.

```php
Capsule::mock('name', fn() => 'szeto');
```

To mock an object, pass an instance of the class as the second argument. This instance will replace any existing
instances bound to the specified key within the capsule.

```php
Capsule::mock('name', new OnBlank('szeto'));
```

Mock with sequence

```php
Capsule::mock(OnBlank::class, new Sequence(
    new OnBlank('szeto'), new OnBlank('joe')
));
capsule()
    ->through(
        fn(OnBlank $name) => expect($name->getKey())->toBe('szeto'),
        fn(OnBlank $name) => expect($name->getKey())->toBe('joe')
    )->run();
```

### Catch

```php
    capsule(
        fn() => throw new Exception('foo'),
        #[Cat(Exception::class)]
        fn($message) => expect($message)->toBe('foo')
    )->run();
```

### Namespace

it can resolve params from namespace, capsules under same namespace can share the value

```php
    capsule()
        ->namespace('some:namespace')
        ->set('name', 'szeto')->run();

    capsule()
        ->namespace('some:namespace')
        ->through(
            fn(string $name) => expect($name)->toBe('szeto')
        )->run();
```

### Append

append function will append the callable to the end of the capsule

```php
    $capsule = capsule()->through(
        #[Setter('name')]
        fn() => 'szeto'
    );

    $name = $capsule->append(
        #[Setter('name')]
        fn($name) => 'joe ' . $name
    )->thenReturn('name');

    expect($name)->toBe('joe szeto');
```

use append combined with namespace

```php
    capsule()->namespace('abc:foo')->append(
        #[Setter('name')]
        fn(string $name) => 'joe ' . $name
    );

    $name = capsule()
        ->namespace('abc:foo')
        ->through(
            #[Setter('name')]
            fn() => 'szeto'
        )->thenReturn('name');

    // now name is joe szeto
```

### massage

when we want to change the value of some variable we will always do something like the following

```php
return capsule()
    ->namespace($namespace)
    ->set($data)
    ->thenReturn($return);
```

so I come up with the idea of massage

```php
return massage($namespace, $data, $return);
```

or

```php
return massage($namespace, $data);
```

but if we want to use this approach, we have to follow some conventions

``` $namespace ``` = ``` someClassname:someFunction:someVariableName ```

and the ``` someVariableName ``` must be the key of the data

for example

```php
massage(Foo::class.':handle:name', ['name' => 'szeto']);
```

this is equivalent to

```php
capsule()
    ->namespace(Foo::class.':handle:name')
    ->set('name', 'szeto')
    ->thenReturn('name');
```
