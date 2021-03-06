This project generates TypeScript code from given Java Classes

## Output format
Java classes are converted to TypeScript interfaces.


### Module (generated output) format
TypeScript has a concept of [internal](http://www.typescriptlang.org/Handbook#modules) and [external](http://www.typescriptlang.org/Handbook#modules-going-external) modules, that have slightly different formats (see the test output for [internal](src/test/resources/java2typescript/jackson/module/DefinitionGeneratorTest.internalModuleFormat.d.ts) vs [external](src/test/resources/java2typescript/jackson/module/DefinitionGeneratorTest.externalModuleFormat.d.ts) module format). By default, internal module format is used (that adds one line before and after content of external module format). To write in the external module format, use the `ExternalModuleFormatWriter` class (see [the test for an example](src/test/java/java2typescript/jackson/module/DefinitionGeneratorTest.java#L85-L97)).


### Ignored methods
When generating TypeScript from Java classes <sup>(actually when Java classes are analysed)</sup>, some methods are excluded:
* Non-public methods
* Methods annotated with @java.beans.Transient
* Java Bean property getters/setters are excluded even if field doesn't exist with exact name (field is generated based on corresponding Java Bean property name instead)
* Methods ignored by the configuration:

```Java
		Configuration conf = new Configuration();
		conf.addIngoredMethod("blacklistedMethod");
```

See the [tests for excluded methods](src/test/java/java2typescript/jackson/module/ExcludedMethodsTest.java#L33-L65) for details.

> Note: You can also extend the configuration and overwrite the [`isIgnoredMethod`](src/main/java/java2typescript/jackson/module/Configuration.java#L44-L49) method to programmatically make the decision (most likely based on method signature: parameters, return type, declaring class, annotations, ...). You may also want to override this method to always return true if you don't want to include any methods in your Typescript definitions.


### Enums
Java enums are converted to TypeScript enums by default,
but TypeSafe enum pattern can be used to force the generation of classes instead of enums (see [the description of the issue](https://github.com/raphaeljolivet/java2typescript/issues/13).
See [this output](src/test/resources/java2typescript/jackson/module/WriterPreferencesTest.enumToEnumPattern.d.ts) from the [test that turns on this preference](src/test/java/java2typescript/jackson/module/WriterPreferencesTest.java#L44) using

```Java
mWriter.preferences.useEnumPattern();
```

### Mapping specific java classes to custom TypeScript types
There are scenarios when you might want to use a different TypeScript type instead of the default Java Type. There are several options for doing this depending how you want it to be done:

##### Renaming Type and emitting it to the output
Logic that determines the TypeScript type based on Java class is implemented using the [TSTypeNamingStrategy](src/main/java/java2typescript/jackson/module/conf/typename/TSTypeNamingStrategy.java) interface. Currently there are two implementations provided out of the box by this library:
1. [SimpleJacksonTSTypeNamingStrategy](src/main/java/java2typescript/jackson/module/conf/typename/SimpleJacksonTSTypeNamingStrategy.java) - The default, uses the Java class name for the TypeScript type, unless the class has an annotation that specifies a custom name (see bellow).
1. [WithEnclosingClassTSTypeNamingStrategy](jackson/module/conf/typename/WithEnclosingClassTSTypeNamingStrategy.java) - extends SimpleJacksonTSTypeNamingStrategy to include the enclosing class name as a prefix (`javaClass.getName()` without package).

###### Renaming Type using annotation on the Java class (SimpleJacksonTSTypeNamingStrategy)
You can use the `@com.fasterxml.jackson.annotation.JsonTypeName("ChangedEnumName")` annotation on the Java type to use a different name in TypeScript output (interface/enum with different name is also generated in the output).
See the [example from the test](src/test/java/java2typescript/jackson/module/DefinitionGeneratorTest.java#L37).

###### Renaming Types using custom (re)naming strategy
To tweak naming TypeScript types for your specific needs, you can provide an implementation of [TSTypeNamingStrategy](src/main/java/java2typescript/jackson/module/conf/typename/TSTypeNamingStrategy.java), such as [SimpleJacksonTSTypeNamingStrategy](src/main/java/java2typescript/jackson/module/conf/typename/SimpleJacksonTSTypeNamingStrategy.java) (or write your own) to [Configuration.setNamingStrategy(namingStrategy)](src/main/java/java2typescript/jackson/module/Configuration.java#L61).

See the test [TypeRenamingWithEnclosingClassTest](src/test/java/java2typescript/jackson/module/TypeRenamingWithEnclosingClassTest.java#L34) and the [expected output of the test](src/test/resources/java2typescript/jackson/module/TypeRenamingWithEnclosingClassTest.twoClassesWithSameName.d.ts#L1).


##### Using different type, not emitting it to the output
One common use-case, could be for instance Joda or Java8 LocalDate or Date or Calendar to TypeScript/JavaScript Date.
See the test [source](src/test/java/java2typescript/jackson/module/CustomTypeDefinitionGeneratorTest.java#L57) and [output](src/test/resources/java2typescript/jackson/module/CustomTypeDefinitionGeneratorTest.classWithCustomTypeFields.d.ts) as an example.

Mapping Java type to TypeScript type is done using:

```Java
new Configuration().addType(CustomDate.class, DateType.getInstance())
```

where `DateType` class specifies the expected output type name.

