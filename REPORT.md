# Spring Boot fixes: cloud session report (2026-10-30)

Two narrow bug fixes are ready on this fork. Each one is on its own branch, based on `upstream/4.0.x`, with a regression test. The fail-before and pass-after Gradle runs are copied verbatim below, and `checkFormat` and `checkstyleMain`/`checkstyleTest` were run for the module. Nothing was opened, edited or commented on upstream.

All `#NNNNN` references in this file are to spring-projects/spring-boot, not to this fork.

| # | Branch | SHA | Base | Suggested PR title |
|---|---|---|---|---|
| 1 | `jsonwriter-named-member-pairs` | `624bbcf3c02f1b82023e9d74dbd740f8a71cfc06` | `4.0.x` | JsonWriter writes invalid JSON when usingPairs is used with a named member |
| 2 | `basic-json-parser-escapes` | `e87247da0db11bfc8a31bc64e8e4d12ea3340d7a` | `4.0.x` | BasicJsonParser does not decode \b, \f, \n, \r, \t and \uXXXX escape sequences |

Both branches are one commit on top of `upstream/4.0.x` at `b7f4c7d4e08` ("Merge pull request #52495 from b84330808"). Each commit is authored and signed off as `Wei-Lun Li <b84330808@gmail.com>` (DCO) and has the `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>` trailer.

There is no ordering dependency between them, and they touch different files. With one spring-boot PR a day and `thread-dump-object-wait` still queued, a reasonable order is `thread-dump-object-wait`, then fix 1, then fix 2.

## Why 4.0.x

- The open milestones are 4.1.3, 4.0.10, 4.2.0-RC2, 4.2.x, 4.1.x, 4.x, 4.0.x and General Backlog (https://github.com/spring-projects/spring-boot/milestones). 4.0.10 is the oldest open patch milestone.
- `upstream/3.5.x` has had no commits since 2026-06-25 ("Next development version (v3.5.16-SNAPSHOT)"), so it is no longer receiving OSS bug fixes.
- `upstream/4.0.x` is at `4.0.10-SNAPSHOT`. The two merged PRs, #52477 and #52495, also went to 4.0.x.
- Both changed files are identical on `4.0.x`, `4.1.x` and `main` (`git diff upstream/4.0.x upstream/main` is empty for them), so the fixes merge forward cleanly.

CONTRIBUTING.adoc and `.github/PULL_REQUEST_TEMPLATE.md` were re-checked. They ask for a Signed-off-by trailer (DCO), Spring JavaFormat, checkstyle, and a brief description that, for a bug, says what is broken and how the change fixes it. No new `.java` files were added.

---

## Fix 1: `JsonWriter` named member with `usingPairs`

- **Branch:** `jsonwriter-named-member-pairs`
- **SHA:** `624bbcf3c02f1b82023e9d74dbd740f8a71cfc06`
- **Upstream base branch:** `4.0.x`
- **Files:** `core/spring-boot/src/main/java/org/springframework/boot/json/JsonWriter.java` (+3 lines) and `core/spring-boot/src/test/java/org/springframework/boot/json/JsonWriterTests.java` (+13 lines)
- **How it was found:** reading the `JsonWriter.Member` Javadoc against its behaviour, then running a standalone repro against the real classes before writing the test.

### PR title

JsonWriter writes invalid JSON when usingPairs is used with a named member

### PR body

````markdown
`JsonWriter.Member.usingPairs(...)` documents that, when it's used with a named member, the pairs are added as a new JSON object:

```json
{
  "name": {
    "p1": 1,
    "p2": 2
  }
}
```

The pairs are currently written straight into the enclosing object instead, which produces invalid JSON:

```java
JsonWriter<Map<String, Object>> writer = JsonWriter.of((members) -> {
	members.add("version", 1);
	members.add("pairs").usingPairs(Map::forEach);
});
writer.writeToString(Map.of("a", "A")); // {"version":1,"pairs":,"a":"A"}
```

`usingExtractedPairs(...)` delegates to `usingPairs(...)`, so it has the same problem. `usingMembers(...)` doesn't, because the `Members` of a named member starts its own object.

This PR updates `JsonWriter.Member` so that the pairs of a named member are written with `JsonValueWriter.writeObject(...)` rather than `writePairs(...)`, consistent with `usingMembers(...)`. Unnamed members are unchanged, so the structured logging formatters, which only use `members.add().usingPairs(...)`, are not affected. The nested pairs get paths such as `pairs.a`, so path filters and name processors apply to them as expected.

Nothing in Spring Boot itself uses `usingPairs` with a named member, but it can be reached through the public API. For example, a `StructuredLoggingJsonMembersCustomizer` that calls `members.add("mdc", ILoggingEvent::getMDCPropertyMap).usingPairs(Map::forEach)` currently produces log lines that aren't valid JSON.

I used Claude Code to write this change.
````

### Diff

```diff
diff --git a/core/spring-boot/src/main/java/org/springframework/boot/json/JsonWriter.java b/core/spring-boot/src/main/java/org/springframework/boot/json/JsonWriter.java
index 1213160d716..5ade9b501c8 100644
--- a/core/spring-boot/src/main/java/org/springframework/boot/json/JsonWriter.java
+++ b/core/spring-boot/src/main/java/org/springframework/boot/json/JsonWriter.java
@@ -650,6 +650,9 @@ public interface JsonWriter<T> {
 		private @Nullable WritableJson getWritableJsonToWrite(@Nullable T extracted, JsonValueWriter valueWriter) {
 			BiConsumer<T, BiConsumer<?, ?>> pairs = this.pairs;
 			if (pairs != null) {
+				if (this.name != null) {
+					return (out) -> valueWriter.writeObject((outPairs) -> pairs.accept(extracted, outPairs));
+				}
 				return (out) -> valueWriter.writePairs((outPairs) -> pairs.accept(extracted, outPairs));
 			}
 			Members<T> members = this.members;
diff --git a/core/spring-boot/src/test/java/org/springframework/boot/json/JsonWriterTests.java b/core/spring-boot/src/test/java/org/springframework/boot/json/JsonWriterTests.java
index 92459c52b78..f2385e70031 100644
--- a/core/spring-boot/src/test/java/org/springframework/boot/json/JsonWriterTests.java
+++ b/core/spring-boot/src/test/java/org/springframework/boot/json/JsonWriterTests.java
@@ -378,6 +378,19 @@ class JsonWriterTests {
 					{"a":"A","b":"B"}""");
 		}
 
+		@Test
+		void usingPairsWithName() {
+			Map<String, Object> map = new LinkedHashMap<>();
+			map.put("a", "A");
+			map.put("b", "B");
+			JsonWriter<Map<String, Object>> writer = JsonWriter.of((members) -> {
+				members.add("version", 1);
+				members.add("pairs").usingPairs(Map::forEach);
+			});
+			assertThat(writer.writeToString(map)).isEqualTo("""
+					{"version":1,"pairs":{"a":"A","b":"B"}}""");
+		}
+
 		@Test
 		void usingPairsWhenAlreadyDeclaredThrowsException() {
 			assertThatIllegalStateException().isThrownBy(() -> JsonWriter.of((
```

### Fail-before (fix reverted, test kept)

Command: `git checkout upstream/4.0.x -- core/spring-boot/src/main/java/org/springframework/boot/json/JsonWriter.java`, then:

```
./gradlew :core:spring-boot:test --tests 'org.springframework.boot.json.JsonWriterTests' --no-scan -Dscan=false -Ddevelocity.scan.disabled=true
```

The output below starts at the test task. Omitted: the `Picked up JAVA_TOOL_OPTIONS` lines (environment proxy settings), Gradle's deprecation and `--warning-mode` boilerplate, blank lines, and a "remote build cache was disabled" notice. The failure message comes from the JUnit XML report.

```
> Task :core:spring-boot:testFixturesClasses UP-TO-DATE
> Task :core:spring-boot:testFixturesJar UP-TO-DATE
> Task :core:spring-boot:compileTestKotlin UP-TO-DATE
> Task :core:spring-boot:compileTestJava UP-TO-DATE
> Task :core:spring-boot:testClasses UP-TO-DATE
> Task :core:spring-boot:test FAILED
JsonWriterTests > MembersTest > usingPairsWithName() FAILED
    org.opentest4j.AssertionFailedError at JsonWriterTests.java:390
87 tests completed, 1 failed
Found test failures in 1 test task:
:core:spring-boot:test
    org.springframework.boot.json.JsonWriterTests$MembersTest > usingPairsWithName()
FAILURE: Build failed with an exception.
* What went wrong:
Execution failed for task ':core:spring-boot:test'.
> There were failing tests. See the report at: file:///home/user/spring-boot/core/spring-boot/build/reports/tests/test/index.html
BUILD FAILED in 15s
36 actionable tasks: 4 executed, 32 up-to-date
=== failure message from test report ===
org.opentest4j.AssertionFailedError: 
expected: "{"version":1,"pairs":{"a":"A","b":"B"}}"
 but was: "{"version":1,"pairs":,"a":"A","b":"B"}"
```

### Pass-after (fix applied, branch HEAD)

Same command, same omissions. The per-suite counts come from the JUnit XML reports.

```
> Task :core:spring-boot:testFixturesClasses UP-TO-DATE
> Task :core:spring-boot:testFixturesJar UP-TO-DATE
> Task :core:spring-boot:compileTestKotlin FROM-CACHE
> Task :core:spring-boot:compileTestJava FROM-CACHE
> Task :core:spring-boot:processTestResources UP-TO-DATE
> Task :core:spring-boot:testClasses UP-TO-DATE
> Task :core:spring-boot:test
BUILD SUCCESSFUL in 7s
36 actionable tasks: 2 executed, 4 from cache, 30 up-to-date
=== test report summary ===
org.springframework.boot.json.JsonWriterTests$MemberPathTests tests=6 skipped=0 failures=0 errors=0
org.springframework.boot.json.JsonWriterTests$MembersTest tests=24 skipped=0 failures=0 errors=0
    <testcase name="usingPairsWithName()" classname="org.springframework.boot.json.JsonWriterTests$MembersTest" time="0.002"/>
org.springframework.boot.json.JsonWriterTests$NameProcessorTests tests=10 skipped=0 failures=0 errors=0
org.springframework.boot.json.JsonWriterTests$PathFilterTests tests=2 skipped=0 failures=0 errors=0
org.springframework.boot.json.JsonWriterTests$StandardWriterTests tests=3 skipped=0 failures=0 errors=0
org.springframework.boot.json.JsonWriterTests$ValueProcessorTests tests=23 skipped=0 failures=0 errors=0
org.springframework.boot.json.JsonWriterTests tests=19 skipped=0 failures=0 errors=0
total tests: 87
```

Broader check on the same commit:

```
./gradlew :core:spring-boot:test --tests 'org.springframework.boot.json.*' --tests 'org.springframework.boot.logging.*' :core:spring-boot:checkFormat :core:spring-boot:checkstyleMain :core:spring-boot:checkstyleTest --no-scan -Dscan=false -Ddevelocity.scan.disabled=true
```

```
> Task :core:spring-boot:checkFormatMain
> Task :core:spring-boot:checkFormatTest
> Task :core:spring-boot:checkFormatTestFixtures
> Task :core:spring-boot:checkFormat
> Task :core:spring-boot:checkstyleTest
> Task :core:spring-boot:checkstyleMain
> Task :core:spring-boot:test
BUILD SUCCESSFUL in 44s
=== json.* + logging.* test results: 872 tests, 0 failures, 0 errors, 1 skipped ===
```

### Duplicate search

| Query | Result |
|---|---|
| https://github.com/spring-projects/spring-boot/issues?q=usingPairs (issues and PRs, any state) | 0 results |
| https://github.com/search?q=repo%3Aspring-projects%2Fspring-boot+usingPairs&type=issues | 0 results |
| https://github.com/spring-projects/spring-boot/issues?q=JsonWriter%20named%20pairs | 0 results |
| https://github.com/spring-projects/spring-boot/issues?q=StructuredLoggingJsonMembersCustomizer%20invalid%20JSON | 0 results |
| https://github.com/spring-projects/spring-boot/issues?q=JsonWriter (both pages, 27 results) | No match. The closest are #52482/#52483/#52477 (NaN, ours), #51464 (`toByteArray` supplementary chars), #47765/#44502 (`Path` StackOverflowError), #45972 (forward slash escaping), #43041 (duplicate names) and #42595 (`WritableJson`). None concern `usingPairs` or named members. |
| `git log upstream/main -- .../json/JsonWriter.java` | The last functional changes were nullability and Javadoc fixes. `getWritableJsonToWrite` has been unchanged since `usingPairs` was introduced. |

---

## Fix 2: `BasicJsonParser` escape sequences

- **Branch:** `basic-json-parser-escapes`
- **SHA:** `e87247da0db11bfc8a31bc64e8e4d12ea3340d7a`
- **Upstream base branch:** `4.0.x`
- **Files:** `core/spring-boot/src/main/java/org/springframework/boot/json/BasicJsonParser.java` (+24/-1) and `core/spring-boot/src/test/java/org/springframework/boot/json/AbstractJsonParserTests.java` (+35)
- **How it was found:** differential testing. 48 hand-written inputs were fed to `BasicJsonParser` and to Jackson's `ObjectMapper`, and the results compared. Before the fix, every escape sequence other than `\"` and `\\` differed. After the fix the only remaining differences are three unrelated, pre-existing ones: a `:` inside a key, leniency for a trailing comma, and an integer larger than `long`. Malformed `\u` escapes (truncated, non-hex, or `\u+041`) now raise `JsonParseException`.

### PR title

BasicJsonParser does not decode \b, \f, \n, \r, \t and \uXXXX escape sequences

### PR body

````markdown
`BasicJsonParser` only decodes the `\"` and `\\` escape sequences. For every other escape sequence it drops the backslash and keeps the next character as-is, so string values are silently corrupted:

| JSON | `BasicJsonParser` | Jackson and Gson |
|---|---|---|
| `{"a":"x\ny"}` | `xny` | `x`, newline, `y` |
| `{"a":"x\ty"}` | `xty` | `x`, tab, `y` |
| `{"a":"caf\u00e9"}` | `cafu00e9` | `café` |

`JsonParserFactory` returns `BasicJsonParser` when neither Jackson 3 nor Gson is on the classpath. That makes it the parser used for, for example, `SPRING_APPLICATION_JSON` in an application without a JSON library. `\uXXXX` escapes are not unusual in practice: Go's `encoding/json`, for instance, escapes `<`, `>` and `&` this way by default.

This PR decodes `\b`, `\f`, `\n`, `\r`, `\t` and `\uXXXX` in the same place that `\"` and `\\` are already handled. That is only when a scalar value is tokenized, so the content of nested maps and lists is still unescaped exactly once (see #51907). The new tests are in `AbstractJsonParserTests`, so they also run against the Jackson and Gson based parsers, which already pass them.

I used Claude Code to write this change.
````

### Diff

```diff
diff --git a/core/spring-boot/src/main/java/org/springframework/boot/json/BasicJsonParser.java b/core/spring-boot/src/main/java/org/springframework/boot/json/BasicJsonParser.java
index f9a809d50e7..0271a0db96d 100644
--- a/core/spring-boot/src/main/java/org/springframework/boot/json/BasicJsonParser.java
+++ b/core/spring-boot/src/main/java/org/springframework/boot/json/BasicJsonParser.java
@@ -6,6 +6,7 @@ package org.springframework.boot.json;
 
 import java.util.ArrayList;
 import java.util.Arrays;
+import java.util.HexFormat;
 import java.util.LinkedHashMap;
 import java.util.List;
 import java.util.Map;
@@ -126,7 +127,12 @@ public class BasicJsonParser extends AbstractJsonParser {
 		while (index < json.length()) {
 			char ch = json.charAt(index);
 			if (tracking.in(Tracked.ESCAPE)) {
-				build.append(ch);
+				if (tracking.in(Tracked.OBJECT, Tracked.LIST)) {
+					build.append(ch);
+				}
+				else {
+					index = unescape(json, index, build);
+				}
 				index++;
 				tracking.set(Tracked.ESCAPE, 0);
 				continue;
@@ -163,6 +169,23 @@ public class BasicJsonParser extends AbstractJsonParser {
 		return list;
 	}
 
+	private static int unescape(String json, int index, StringBuilder build) {
+		char ch = json.charAt(index);
+		switch (ch) {
+			case 'b' -> build.append('\b');
+			case 'f' -> build.append('\f');
+			case 'n' -> build.append('\n');
+			case 'r' -> build.append('\r');
+			case 't' -> build.append('\t');
+			case 'u' -> {
+				build.append((char) HexFormat.fromHexDigits(json, index + 1, index + 5));
+				return index + 4;
+			}
+			default -> build.append(ch);
+		}
+		return index;
+	}
+
 	private static final class Tracking {
 
 		private final int[] counts = new int[Tracked.values().length];
diff --git a/core/spring-boot/src/test/java/org/springframework/boot/json/AbstractJsonParserTests.java b/core/spring-boot/src/test/java/org/springframework/boot/json/AbstractJsonParserTests.java
index e05532969c0..b61901d9bdd 100644
--- a/core/spring-boot/src/test/java/org/springframework/boot/json/AbstractJsonParserTests.java
+++ b/core/spring-boot/src/test/java/org/springframework/boot/json/AbstractJsonParserTests.java
@@ -259,6 +259,28 @@ abstract class AbstractJsonParserTests {
 		assertThat(map).containsEntry("a", "alpha\"");
 	}
 
+	@Test
+	void escapeControlCharacters() {
+		String input = """
+				{
+				  "a": "alpha\\b\\f\\n\\r\\tbravo"
+				}
+				""";
+		Map<String, Object> map = this.parser.parseMap(input);
+		assertThat(map).containsEntry("a", "alpha\b\f\n\r\tbravo");
+	}
+
+	@Test
+	void escapeUnicode() {
+		String input = """
+				{
+				  "a": "alpha\\u0026bravo"
+				}
+				""";
+		Map<String, Object> map = this.parser.parseMap(input);
+		assertThat(map).containsEntry("a", "alpha&bravo");
+	}
+
 	@Test
 	void escapeBackslashInNestedMap() {
 		String input = """
@@ -285,6 +307,19 @@ abstract class AbstractJsonParserTests {
 		assertThat(map).extractingByKey("outer", InstanceOfAssertFactories.MAP).containsEntry("a", "alpha\"");
 	}
 
+	@Test
+	void escapeUnicodeInNestedMap() {
+		String input = """
+				{
+				  "outer": {
+				    "a": "alpha\\u0026bravo"
+				  }
+				}
+				""";
+		Map<String, Object> map = this.parser.parseMap(input);
+		assertThat(map).extractingByKey("outer", InstanceOfAssertFactories.MAP).containsEntry("a", "alpha&bravo");
+	}
+
 	@Test
 	void escapeBackslashInNestedList() {
 		String input = """
```

### Fail-before (fix reverted, test kept)

Command: `git checkout upstream/4.0.x -- core/spring-boot/src/main/java/org/springframework/boot/json/BasicJsonParser.java`, then:

```
./gradlew :core:spring-boot:test --tests 'org.springframework.boot.json.*JsonParserTests' --no-scan -Dscan=false -Ddevelocity.scan.disabled=true
```

Same omissions as fix 1. The `escapeControlCharacters()` message is shown exactly as it appears in the JUnit XML report: the expected value's control characters are rendered there, with the backspace and form feed showing up as `?`. Note that only `BasicJsonParserTests` fails: the same three tests pass for `JacksonJsonParserTests` and `GsonJsonParserTests` (137 tests in total, 3 failed).

```
> Task :core:spring-boot:testFixturesClasses UP-TO-DATE
> Task :core:spring-boot:testFixturesJar UP-TO-DATE
> Task :core:spring-boot:compileTestKotlin UP-TO-DATE
> Task :core:spring-boot:compileTestJava UP-TO-DATE
> Task :core:spring-boot:processTestResources UP-TO-DATE
> Task :core:spring-boot:testClasses UP-TO-DATE
> Task :core:spring-boot:test
BasicJsonParserTests > escapeUnicodeInNestedMap() FAILED
    java.lang.AssertionError at AbstractJsonParserTests.java:320
BasicJsonParserTests > escapeControlCharacters() FAILED
    java.lang.AssertionError at AbstractJsonParserTests.java:270
BasicJsonParserTests > escapeUnicode() FAILED
    java.lang.AssertionError at AbstractJsonParserTests.java:281
OpenJDK 64-Bit Server VM warning: Sharing is only supported for boot loader classes because bootstrap classpath has been appended
> Task :core:spring-boot:test FAILED
137 tests completed, 3 failed
Found test failures in 1 test task:
:core:spring-boot:test
    org.springframework.boot.json.BasicJsonParserTests > escapeControlCharacters()
    org.springframework.boot.json.BasicJsonParserTests > escapeUnicode()
    org.springframework.boot.json.BasicJsonParserTests > escapeUnicodeInNestedMap()
FAILURE: Build failed with an exception.
* What went wrong:
Execution failed for task ':core:spring-boot:test'.
> There were failing tests. See the report at: file:///home/user/spring-boot/core/spring-boot/build/reports/tests/test/index.html
BUILD FAILED in 9s
36 actionable tasks: 2 executed, 2 from cache, 32 up-to-date
=== failure messages from BasicJsonParserTests report ===
-- escapeUnicodeInNestedMap()
java.lang.AssertionError: [Extracted: outer] 
Expecting map:
  {"a"="alphau0026bravo"}
to contain entries:
  ["a"="alpha&bravo"]
but the following map entries had different values:
  ["a"="alphau0026bravo" (expected: "alpha&bravo")]
-- escapeControlCharacters()
java.lang.AssertionError: 
Expecting map:
  {"a"="alphabfnrtbravo"}
to contain entries:
  ["a"="alpha??
	bravo"]
but the following map entries had different values:
  ["a"="alphabfnrtbravo" (expected: "alpha??
	bravo")]
-- escapeUnicode()
java.lang.AssertionError: 
Expecting map:
  {"a"="alphau0026bravo"}
to contain entries:
  ["a"="alpha&bravo"]
but the following map entries had different values:
  ["a"="alphau0026bravo" (expected: "alpha&bravo")]
```

### Pass-after (fix applied, branch HEAD)

Same command with `--rerun`. The per-suite counts and new test cases come from the JUnit XML reports.

```
> Task :core:spring-boot:testFixturesClasses UP-TO-DATE
> Task :core:spring-boot:testFixturesJar UP-TO-DATE
> Task :core:spring-boot:compileTestKotlin UP-TO-DATE
> Task :core:spring-boot:compileTestJava UP-TO-DATE
> Task :core:spring-boot:processTestResources UP-TO-DATE
> Task :core:spring-boot:testClasses UP-TO-DATE
> Task :core:spring-boot:test
OpenJDK 64-Bit Server VM warning: Sharing is only supported for boot loader classes because bootstrap classpath has been appended
BUILD SUCCESSFUL in 8s
36 actionable tasks: 1 executed, 35 up-to-date
=== test report summary ===
org.springframework.boot.json.BasicJsonParserTests tests=46 skipped=0 failures=0 errors=0
    <testcase name="escapeControlCharacters()" classname="org.springframework.boot.json.BasicJsonParserTests" time="0.001"/>
    <testcase name="escapeUnicode()" classname="org.springframework.boot.json.BasicJsonParserTests" time="0.001"/>
    <testcase name="escapeUnicodeInNestedMap()" classname="org.springframework.boot.json.BasicJsonParserTests" time="0.062"/>
org.springframework.boot.json.GsonJsonParserTests tests=45 skipped=0 failures=0 errors=0
    <testcase name="escapeControlCharacters()" classname="org.springframework.boot.json.GsonJsonParserTests" time="0.001"/>
    <testcase name="escapeUnicode()" classname="org.springframework.boot.json.GsonJsonParserTests" time="0.0"/>
    <testcase name="escapeUnicodeInNestedMap()" classname="org.springframework.boot.json.GsonJsonParserTests" time="0.001"/>
org.springframework.boot.json.JacksonJsonParserTests tests=46 skipped=0 failures=0 errors=0
    <testcase name="escapeControlCharacters()" classname="org.springframework.boot.json.JacksonJsonParserTests" time="0.001"/>
    <testcase name="escapeUnicode()" classname="org.springframework.boot.json.JacksonJsonParserTests" time="0.001"/>
    <testcase name="escapeUnicodeInNestedMap()" classname="org.springframework.boot.json.JacksonJsonParserTests" time="0.003"/>
```

Broader check on the same commit:

```
./gradlew :core:spring-boot:test --tests 'org.springframework.boot.json.*' --tests 'org.springframework.boot.support.SpringApplicationJsonEnvironmentPostProcessorTests' --tests 'org.springframework.boot.cloud.*' :core:spring-boot:checkFormat :core:spring-boot:checkstyleMain :core:spring-boot:checkstyleTest --no-scan -Dscan=false -Ddevelocity.scan.disabled=true
```

```
> Task :core:spring-boot:checkFormatMain
> Task :core:spring-boot:checkFormatTest UP-TO-DATE
> Task :core:spring-boot:checkFormatTestFixtures UP-TO-DATE
> Task :core:spring-boot:checkFormat
> Task :core:spring-boot:checkstyleTest
> Task :core:spring-boot:checkstyleMain
> Task :core:spring-boot:test
BUILD SUCCESSFUL in 36s
=== json.* + SpringApplicationJsonEnvironmentPostProcessorTests + cloud.* : 326 tests, 0 failures, 0 errors, 1 skipped ===
```

### Duplicate search

| Query | Result |
|---|---|
| https://github.com/spring-projects/spring-boot/issues?q=BasicJsonParser (both pages, 42 results, all closed) | No match. The related, already-fixed issues are #51907/#51908/#51909 (escapes in nested elements, fixed by Andy Wilkinson in `f6aa92821da`), #51968 (escaped quote, duplicate), #52317/#52318/#52179 (list whitespace), #52325/#52326/#52182 (braces in strings), #48998 (":" in values, closed as invalid because values with ":" already work) and #14348/#14521 (2018, escaped double quote). None cover `\n`, `\t` or `\uXXXX`. |
| https://github.com/spring-projects/spring-boot/issues?q=BasicJsonParser%20unicode | 0 results |
| https://github.com/spring-projects/spring-boot/issues?q=json%20escape%20newline | 0 results |
| `git log upstream/main -- .../json/BasicJsonParser.java` | The latest commits are `5024e5d1c0e` and `85c07e91f54` (braces, 2026-10-21), `2cdf1990a3f` (list whitespace), `86795008d1a` (escaped quote in nested elements) and `f6aa92821da` (gh-51907). None decode these escapes. |

---

## Full open-PR check (both fixes)

Source: https://github.com/spring-projects/spring-boot/pulls?q=is%3Apr+is%3Aopen, both pages, fetched at the start of the session and again before writing this report. The list was the same both times: 28 open PRs. The GitHub API isn't reachable from this session, so the web listing was used. None of them touch `JsonWriter`, `JsonValueWriter` or `BasicJsonParser`.

- [#52491](https://github.com/spring-projects/spring-boot/pull/52491) Provide customizer callback for embedded LDAP's InMemoryDirectoryServerConfig (Eymenonar)
- [#52466](https://github.com/spring-projects/spring-boot/pull/52466) Generate Logback configurator during AOT processing (ceki)
- [#52456](https://github.com/spring-projects/spring-boot/pull/52456) Document starter parent plugin configuration (vanvo)
- [#52436](https://github.com/spring-projects/spring-boot/pull/52436) Add property to configure Elasticsearch's response timeout (beyondwin)
- [#52433](https://github.com/spring-projects/spring-boot/pull/52433) Apply default ports to Redis connections (noojung)
- [#52398](https://github.com/spring-projects/spring-boot/pull/52398) Cache compiled Mustache templates in Spring MVC (vanvo)
- [#52395](https://github.com/spring-projects/spring-boot/pull/52395) Size the nested jar inflater buffer from the compressed size (gtiwari333)
- [#52380](https://github.com/spring-projects/spring-boot/pull/52380) JSON parse failures no longer produce DockerOutputParseExceptions (zhoubiao188)
- [#52362](https://github.com/spring-projects/spring-boot/pull/52362) Include nested paths in structured logging JSON include filters (zhoubiao188)
- [#52349](https://github.com/spring-projects/spring-boot/pull/52349) Add R2DBC jOOQ auto-configuration (GeorgePap-719)
- [#52281](https://github.com/spring-projects/spring-boot/pull/52281) Support SSL bundle reloading with Jetty (liotur)
- [#52279](https://github.com/spring-projects/spring-boot/pull/52279) Add SSL bundle support to ActiveMQ auto-configuration (liotur)
- [#52241](https://github.com/spring-projects/spring-boot/pull/52241) Add mail sender service connection support (vanvo)
- [#52233](https://github.com/spring-projects/spring-boot/pull/52233) Clean up logging system after beans have been destroyed (yusufyildiz34)
- [#52163](https://github.com/spring-projects/spring-boot/pull/52163) Logging can be lost when a context is stopped and started again (challa)
- [#52160](https://github.com/spring-projects/spring-boot/pull/52160) Add Jackson 3 support to Couchbase auto-configuration (emilienbev)
- [#52159](https://github.com/spring-projects/spring-boot/pull/52159) Support non-RSA keys for SAML2 signing credentials (liotur)
- [#52120](https://github.com/spring-projects/spring-boot/pull/52120) Reject unsupported *_FILE variables in Docker Compose support (DDINGJOO)
- [#52118](https://github.com/spring-projects/spring-boot/pull/52118) Support @ServiceConnection @Bean methods in AOT-processed tests (seonwooj0810)
- [#51936](https://github.com/spring-projects/spring-boot/pull/51936) Use Jackson 3 by default in the Jersey starter (khopland)
- [#51930](https://github.com/spring-projects/spring-boot/pull/51930) Add symbol processor for configuration metadata that shares the implementation of the annotation processor (salatmaster)
- [#51903](https://github.com/spring-projects/spring-boot/pull/51903) Support configuring the embedded LDAP listen address (Wanbinyu)
- [#51853](https://github.com/spring-projects/spring-boot/pull/51853) Bind immutable collections and maps by default (ABin-Huang)
- [#51556](https://github.com/spring-projects/spring-boot/pull/51556) Index indexed children rather than scanning per element (tommyk-gears)
- [#51286](https://github.com/spring-projects/spring-boot/pull/51286) Add an @XmlTest test slice similar to @JsonTest (basteez)
- [#50969](https://github.com/spring-projects/spring-boot/pull/50969) Restore default JUL root handler on Logback cleanup (rsrizvi)
- [#50280](https://github.com/spring-projects/spring-boot/pull/50280) Allow configuring an SSLBundle to use when making JWKS requests (bbbbooo)
- [#34900](https://github.com/spring-projects/spring-boot/pull/34900) Short circuit @Value injection when using a source already covered by ConfigurationPropertySourcesPropertySource (terminux)

The JSON- and logging-adjacent ones were looked at specifically:

- #52362 changes structured logging `include` filtering. It is about `StructuredLoggingJsonPropertiesJsonMembersCustomizer`, not `JsonWriter.Member`, and it's the item listed as already taken.
- #52380 is Docker Compose JSON parse exceptions, not `BasicJsonParser`.
- #52233, #52163 and #50969 are logging system lifecycle changes.

---

## Other leads (not fixed here)

These came up during the session. They are recorded so a later session can pick them up. Only the first has been checked for duplicates, and only lightly.

1. **`spring.config.import` rewrites the file extension's case.**
   - `StandardConfigDataLocationResolver.getLoadableFileExtension` matches with `StringUtils.endsWithIgnoreCase(file, "yml")`, with no leading dot, and returns the loader's lowercase extension. `getReferencesForFile` then rebuilds the location from that lowercase extension.
   - As a result, `spring.config.import=file:./APP.YML` is looked up as `APP.yml`, which fails on a case-sensitive file system, or is silently skipped with `optional:`.
   - An extensionless `settings_yml` is resolved as `settings.yml`.
   - This was reproduced by a research subagent against the real classes, and the code path was confirmed by reading it. Not yet checked: duplicates, and whether case-insensitive matching is intentional.
2. **WebFlux `httpexchanges` `remoteAddress` format differs from servlet.**
   - `RecordableServerHttpRequest.getRemoteAddress` uses `InetAddress.toString()`, which gives `/127.0.0.1`. The servlet version uses `getRemoteAddr()`, which gives `127.0.0.1`.
   - With forwarded headers, the WebFlux address is unresolved, so the WebFlux value becomes `null`.
   - The existing tests assert the current form, so fixing it is a visible behaviour change.
3. **`StandardStackTracePrinter.withMaximumLength`, the weak lead from before, is a real but small bug.**
   - `Output` reserves 3 characters for `...` up front, so output whose total length is between `maximumLength - 2` and `maximumLength` is truncated with `...` even though it fits.
   - `maximumLength` values of 1 to 3 print nothing.
   - Contract: "use ellipses to truncate output longer than the specified length".
   - It's low value, so it was not taken.
4. **`/env` with nested `CompositePropertySource`.** `EnvironmentEndpoint` builds nested names with `source.getName() + ":"` and drops the outer prefix. It's an edge case.

## Build environment notes

- The build requires JDK 25 (`BUILD_JAVA_VERSION = 25`), so `openjdk-25-jdk-headless` was installed with apt and `JAVA_HOME` pointed at it.
- `repo.maven.apache.org` returned intermittent HTTP 429 to this container. A local, uncommitted `~/.gradle/init.d` script pointed Maven Central at Google's Central mirror (`maven-central.storage-download.googleapis.com`), and a copy went into `buildSrc`'s TestKit Gradle home.
- Every Gradle invocation used `--no-scan -Dscan=false -Ddevelocity.scan.disabled=true`, so no build scan was published.
