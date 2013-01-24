# FM-Parameters

FM-Parameters is a series of custom functions for creating and parsing serialized data in FileMaker. These are most frequently used for passing multiple pieces of data in script parameters and results. This repository is an incomplete work in progress based on the recommended script parameter interface of [FileMakerStandards.org][1].

[1]: http://filemakerstandards.org/pages/viewpage.action?pageId=557462 "FileMakerStandards.org: Script Parameter Interface"

## The format

In theory, the data serialization format is irrelevant to how these functions should behave relative to each other. Changing the data format used by the functions should not require any changes to their use. In this version, the functions manipulate data stored in a format that matches the variable-declaration format of FileMaker's Let () function. This is so that the format will appear familiar to FileMaker developers without additional background knowledge. This mirrors the relationship between JavaScript and JSON.

## Core Functions

These functions implement the basics of adding data to and retrieving data from dictionary data structures.

### # ( name ; value )

The # ( name ; value ) function creates a Let notation name-value pair. A dictionary data structure can be created by concatenating several calls to #() as if they were plain text. Let notation also supports nested dictionaries.

	# ( "name" ; "value" )
	& # ( "outerName" ;
		# ( "innerName" ; "inner value" )
	)

Named values can be over-written or effectively erased by concatenating a call to #() to the end of a dictionary using the same name and a different value. This works because the #Get and #Assign functions will always respect the *last* instance of a name-value pair in a dictionary.

	# ( "name" ; "value" )
	& # ( "foo" ; "bar" )
	& # ( "name" ; "new value" )
	& # ( "foo" ; "" )	// Assigning empty value effectively deletes "foo"

This last-value-wins behavior can also be used to set default values for optional parameters.

	# ( "optionalParameter" ; "default value" )
	& Get ( ScriptParameter )

By placing the defaults before the actual parameters, any values set by the actual script parameter will override the defaults.

Past versions of the # ( name ; value ) function would accept global variable names for the name parameter. This version of the function will convert the global variable to a local variable. I believe it's better that the code assigning data to variables, rather than the code serializing the data, should decide what scope of variables to use. Developers interested in setting global variables from serialized data should use the #AssignGlobal, #Filter, and #Get functions to achieve similar behaviors.

### #Assign ( parameters )

The #Assign functions parse a dictionary into local script variables. The name from each name-value pair is used as the variable name, and the value from each pair is set to that variable's value.

	#Assign ( # ( "name" ; "value" ) )	// variable $name assigned "value"

The #Assign function returns True (1) if there was no error detected while assigning the values to variables, and returns False (0) otherwise. If there was an error detected, FileMaker's error code is assigned to the $#Assign.error variable.

Some legacy versions of the # ( name ; value ) function allow developers to set names intended for global variables. The # ( name ; value ) function defined here will not allow that, but legacy code may still include name-value pairs targeted at global variables. This version of the #Assign function will assign the values to local variables instead.

### #Get ( parameters ; name )

The #Get function returns a named value from a dictionary.

	#Get ( # ( "name" ; "value" ) ; "name" )	// = "value"

Unlike the #Assign functions, #Get will not affect any variables, which can be useful when a named value only needs to be used in one calculation, or to assign a named value to a variable with a different name.

## Utility Functions

These functions are less fundamental to working with dictionary data structures than the core functions, but experience has demonstrated that this functionality is indispensable in practical applications.

### #AssignGlobal ( parameters )

The #AssignGlobal function parses a dictionary into global variables.

	#AssignGlobal ( # ( "name" ; "value" ) )	// variable $$name assigned "value"

This approach to setting global variables is preferred over other methods that allow the dictionary itself to define what values should be assigned to globals. It makes more sense that the code that assigns the variables should have discretion over what scope of variable gets assigned.

The #AssignGlobal function return True (1) if there was no error detected while assigning the values to variables, and returns False (0) otherwise. If there was an error detected, FileMaker's error code is assigned to the $#AssignGlobal.error variable.

### #GetNameList ( parameters )

The #GetNameList function returns a return-delimited list of (distinct) names from name-value pairs in the parameters.

	#GetNameList (
		# ( "name" ; "value" )
		& # ( "foo" ; Random )
		& # ( "bar" ; Random )
		& # ( "name" ; "another value" )
	)	// = "name¶foo¶bar"

### #Filter ( parameters ; filterParameters )

The #Filter function returns a Let format dictionary containing only those name-value pairs where the name is included in the return-delimited list filterParameters.

	#Assign ( #Filter (
		# ( "name" ; "value" )
		& # ( "foo" ; "bar" );
		List ( "name" ; "otherName" )
	) )	// variable $name assigned "value"; $foo and $otherName are unaffected

This function can prevent an "injection" of unexpected variables that might cause problems.

### #Remove ( parameters ; removeParameters )

The #Remove function returns a Let format dictionary containing only those name-value pairs where the name is not included in the return-delimited list removeParameters. This is complementary to the #Filter function. As illustrated in the documentation for the # ( name ; value ) function, a similar effect can be achieved by concatenating an empty value for a name to a dictionary, but there is an important difference when using the resulting dictionary with the #Assign function.

	Set Variable [$name; Value:"value"]
	Set Variable [$~; Value:#Assign ( #Remove ( $dictionary ; "name" ) )]
	# $name = "value"
	Set Variable [$~; Value:#Assign ( $dictionary & # ( "name" ; "" ) )]
	# $name = ""

The #Remove function is also useful for "compacting" a dictionary after some values are no longer needed.

### ScriptOptionalParameterList ( scriptNameToParse )

Parses a script name, returning a return-delimited list of optional parameters for that script, in the order they appear in the script name. This function assumes that the script name conforms to the FileMakerStandards.org [naming convention for scripts][2]. This is useful to generate the argument used by the #Filter function to restrict variable assignment to parameters actually accepted by a script. As with the ScriptRequiredParameterList function, when the scriptNameToParse parameter is empty, the function will use the current script name.

	#Assign ( #Filter (
		Get ( ScriptParameter );
		ScriptRequiredParameterList ( "" ) & ScriptOptionalParameterList ( "" )
	) )

### ScriptRequiredParameterList ( scriptNameToParse )

Parses a script name, returning a return-delimited list of parameters required for that script, in the order they appear in the script name. This function assumes that the script name conforms to the FileMakerStandards.org [naming convention for scripts][2]. This is useful to generate the argument used by the VerifyVariablesNotEmpty function to validate that all required parameters have values. When the scriptNameToParse parameter is empty, the function will use the current script name:

	ScriptRequiredParameterList ( "" ) = ScriptRequiredParameterList ( Get ( ScriptName ) )

[2]: http://filemakerstandards.org/display/cs/Script+naming "FileMakerStandards.org: Script naming"

### VerifyVariablesNotEmpty ( nameList )

Returns True (1) if each of the parameters in parameterNameList has been assigned to a non-empty local script variable of the same name. Returns False (0) if any variable defined by nameList is empty.

	VariablesNotEmpty ( List ( "parameter1" ; "parameter2" ) )

This is useful for verifying that each parameter required by a script has been successfully assigned before proceeding.

## Legacy functions

These functions are implemented for the sake of backwards compatibility in those solutions using the current FileMakerStandards.org functions. These functions are all equivalent to simple combinations of other functions. These functions may be easier to type than the equivalent combinations of other functions, but tools like [TextExpander][], [Breevy][], and [Clip Manager][] make this a moot point. This approach also makes the resulting code more transparent about what logic is being applied to what data inputs, making the code more readable, especially to developers unfamiliar with the functions.

[TextExpander]: http://smilesoftware.com/TextExpander/index.html
[Breevy]: http://www.16software.com/breevy/
[Clip Manager]: http://www.myfmbutler.com/index.lasso?p=422

### #AssignScriptParameters

The #AssignScriptParameters function will assign all named values in the script parameter to local script variables of the same name. If any parameters indicated as required by the script name are empty, the function returns False (0); the function returns True (1) otherwise.

	If [not #AssignScriptParameters]
		Exit Script [Result:# ( "error" ; 10 )	// Requested data is missing]
	End If

A combination of the #Assign, VariablesNotEmpty, and ScriptRequiredParameterList functions is the preferred way to replicate this behavior:

	Set Variable [$ignoreMe; Value:#Assign ( Get ( ScriptParameter ) )]
	If [not VerifyVariablesNotEmpty ( ScriptRequiredParameterList ( Get ( ScriptName ) ) )]
		Exit Script [Result:# ( "error" ; 10 )	// Requested data is missing]
	End If

This approach enables greater flexibility in defining what variables are required and how those variables are assigned.

### #AssignScriptResults

This function is exactly equivalent to this calculation:

	#Assign ( Get ( ScriptResult ) )

Using this calculation instead of this function is preferred.

### #GetScriptParameter ( name )

This function is exactly equivalent to this calculation:

	#Get ( Get ( ScriptParameter ) ; name )

Using this calculation instead of this function is preferred.

### #GetScriptResult ( name )

This function is exactly equivalent to this calculation:

	#Get ( Get ( ScriptResult ) ; name )

Using this calculation instead of this function is preferred.

## Roadmap

The core and utility functions accomplish what most of us need most of the time, but this is some nice-to-have functionality we're considering implementing in the future.

### #List ( value )

Encode a value so it can be stored as a single value in a FileMaker List.

### #ListGet ( listOfValues ; valueNumber )

Retrieve a value from a list created with the #List custom function.

## Acknowledgements

The Let format was inspired by an example in FileMaker's [documentation for the Let() function][3].

[3]: http://www.filemaker.com/help/html/func_ref3.33.15.html "FileMaker, Inc.: Let"

The dictionary functions by Six Fried Rice (mentioned in [introductory][4] and [supplemental][5] posts) were a significant inspiration for the behaviors of some functions and the "#" prefix notation.

[4]: http://sixfriedrice.com/wp/passing-multiple-parameters-to-scripts-advanced/ "Six Fried Rice: Passing Multiple Parameters to Scripts - Advanced"
[5]: http://sixfriedrice.com/wp/filemaker-dictionary-functions/ "Six Fried Rice: FileMaker Dictionary Functions"

The commenters at [FileMakerStandards.org][1] were instrumental in refining the function interfaces.

## License

Anyone may do anything with this software. There is no warranty.