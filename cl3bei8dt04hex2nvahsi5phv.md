## An approach to error handling with null in C#

**Ever since C# 8, we've had the possibility to use [Nullable Reference Types](https://docs.microsoft.com/en-us/dotnet/csharp/nullable-references), and the hope was that by controlling where we pass null to and from functions we should be able to get rid of null handling all together.**

**Instead I feel that we've got code that looks like it's questioning it's own existence by the multitude of question marks.**

To give a little bit of context, I was looking into [Railway Oriented Programming](https://fsharpforfunandprofit.com/rop/), and wondered how one could integrate those ideas into C# without forcing it. The idea is that you conceptually form two railway tracks in your code. There's a happy path where everything works fine, and there's a parallel track where you're redirected when an error is encountered.

So, in a functional programming language you'd implement that with a "Maybe" or "Option" monad. It could be viewed as a collection that is either empty or contains exactly one element. That sounds cool, so let's create a collection that can contain maximum one element and that implements ```
IEnumerable<T>
```. Now you can run LINQ on it... Except, there's not a good way to signal an error, as LINQ is working element by element. And also, every normal LINQ extension method would return an ```
IEnumerable<T>
``` and not my desired "Maybe" collection.

I was thinking about this for a while when I heard [an interview with Mads Torgersen](https://nodogmapodcast.bryanhogan.net/124-mads-torgersen-c-8/) about his vision for C# 9 and beyond where he envisioned support for [Discriminated Unions](https://docs.microsoft.com/en-us/dotnet/fsharp/language-reference/discriminated-unions). DU:s would make this a lot easier to implement. DU:s are still nowhere to be seen, but Mads also mentioned (48 minutes in) that Nullable Reference Types could be seen as a DU that could either be null or something else.

So I came up with this. You don't have to throw exceptions on null. Instead of avoiding null we can embrace it.

```
#nullable enable

public static class NullableExtensions
{
	public static T2? Map<T1, T2>(this T1? nullable, Func<T1, T2?> func) =>
		nullable switch
		{
			null => default(T2),
			_ => func(nullable)
		};
}
``` 

The interesting thing with this is that you can basically ignore null.


```
int? empty = null;
Console.WriteLine(empty.Map(s => s + 1)); // Writes "null"

int? something = 1;
Console.WriteLine(something.Map(s => s + 1)); // Writes "2"
``` 

So to bring it to the conclusion, you can chain multiple calls to ```
.Map()
``` without having to deal with null.


```
var result = something
	.Map(s => s + 1)
	.Map(s => $"The result is {s}");
	
Console.WriteLine(result); // Writes "The result is 2"
``` 

Or, you can signal an error early in the chain by returning null.


```
var result2 = something
	.Map(s => s < 1 ? s + 1: null)
	.Map(s => $"The result is {s}");
	
Console.WriteLine(result2); // Writes "null"
```

If you want to use this in an ASP.Net controller you might want an extension method that can convert this into an ```
IActionResult
```. That could look somehting like this.


```
#nullable enable

public static class NullableExtensions
{
	public static T2? Map<T1, T2>(this T1? nullable, Func<T1, T2?> func) =>
		nullable switch
		{
			null => default(T2),
			_ => func(nullable)
		};
		
	public static IActionResult AsActionResult<T>(this T? nullable, Func<IActionResult> onNull) =>
        nullable switch
        {
            null => onNull(),
            _ => new OkObjectResult(nullable)
        };
}
``` 

Then you could write your endpoint like this.

```
[HttpGet("{sessionId}")]
public IActionResult GetStatus(string sessionId) =>
    repository.GetStatus(sessionId)
    .Map(s => s.MapToApiModel())
    .AsActionResult(onNull: NotFound);
```

Even if ```
repository.GetStatus(sessionId)
``` returns ```
null
```, you don't have to explicitly handle it. Instead you pass the result to the next function using .Map().

The extension method ```
.AsActionResult(onNull: NotFound)
``` will convert the result to a HTTP response 200 on success, and to 404 if it receives ```
null
```.

Please let me know if you find this useful. :)
