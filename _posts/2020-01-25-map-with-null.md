---
title: "How implementations of java.util.Map interact with NULL keys and values?"
date: 2020-01-25
categories: java
excerpt: "Unexpected behaviour of Collectors.toMap() with NULL values"
---

Dictionary (or Map in the world of Java) is truly one of the most powerful data structures in programming.   
A good understanding of its work is an essential skill for every programmer. However, sometimes your code performs in a very strange way and you don't understand why.   

This happened to me and my colleague a few days ago, so I'd like to share this interesting case with you.

## How java.util.Map works with NULL
Let's explore some of the most popular implementations of java.util.Map interface:

### HashMap
HashMap in Java is the most flexible and widely used implementation and it works perfectly with both *null* keys and values.  
HashMap has several LinkedList in its *buckets*. And after calculating *hashCode()* on the key this structure it takes a decision in which bucket it will hold the *Entry (key+value)*.

**Keys** - Instead of calculating *hashCode()* on *null*, HashMap treats *null*-key as a special case and uses "0" instead, so the value of *null*-key is always in the bucket "0".

**Values** - You are also able to use *null* as a value. But be careful if you have some logic on this because you will get *null* either if you have *null*-value or the key is absent. Don't forget to check whether the key is present or not in a map with *containsKey(Object key)* method.
```java
@Test
public void hashMapNullKeyTest() {
Map<String, String> hashMap = new HashMap<>();
    hashMap.put(null, "1");
    Assert.assertEquals(1, hashMap.size());
    Assert.assertTrue(hashMap.containsKey(null));
    Assert.assertEquals("1", hashMap.get(null));
}

@Test
public void hashMapNullValueTest() {
    Map<String, String> hashMap = new HashMap<>();
    hashMap.put("key", null);
    Assert.assertEquals(1, hashMap.size());
    Assert.assertTrue(hashMap.containsKey("key"));
    Assert.assertNull(hashMap.get("key"));
    // Be careful with this case
    Assert.assertFalse(hashMap.containsKey("absentKey"));
    Assert.assertNull(hashMap.get("absentKey"));
}
```
### LinkedHashMap
LinkedHashMap is more prefferable if you want near-HashMap performance and insertion-order iteration.  
In the case of *null*-keys and values it behaves the same as HashMap:

```java
@Test
public void linkedHashMapNullKeyTest() {
    Map<String, String> linkedHashMap = new LinkedHashMap<>();
    linkedHashMap.put(null, "1");
    Assert.assertEquals(1, linkedHashMap.size());
    Assert.assertTrue(linkedHashMap.containsKey(null));
    Assert.assertEquals("1", linkedHashMap.get(null));
}

@Test
public void linkedHashMapNullValueTest() {
    Map<String, String> linkedHashMap = new LinkedHashMap<>();
    linkedHashMap.put("key", null);
    Assert.assertEquals(1, linkedHashMap.size());
    Assert.assertTrue(linkedHashMap.containsKey("key"));
    Assert.assertNull(linkedHashMap.get("key"));
    // Be careful with this case
    Assert.assertFalse(linkedHashMap.containsKey("absentKey"));
    Assert.assertNull(linkedHashMap.get("absentKey"));
}
```

### TreeMap
TreeMap, however, is a different beast. It uses a **red-black tree** which is a kind of self-balancing binary search tree. So its behavior with *null* is non-identical.

**Keys** - Here comes the difference. TreeMap doesn't allow us to use *null* as a key:
```java
@Test
public void treeMapNullKeyTest() {
    Map<String, String> treeMap = new TreeMap<>();
    treeMap.put(null, "value");
    Assert.assertEquals("value", treeMap.get(null));
}
```
...and on the line with *put()* call we get *NullPointerException*:
```
java.lang.NullPointerException
	at java.base/java.util.TreeMap.compare(TreeMap.java:1291)
	at java.base/java.util.TreeMap.put(TreeMap.java:536)
	at io.github.vergl.maps.MapTest.treeMapNullKeyTest(MapTest.java:52)
```
In *compare()* method TreeMap calls *compareTo()* on key, which leads us to NPE.

<p>
In older JDK versions (before 1.7) there was no call of <i>compareTo()</i> for the first key in the map so you were able to add <i>null</i> key as the first element of TreeMap. But after that, each operation (except <i>size()</i> and <i>clear()</i>) was ending with NPE.
</p>{: .notice--info}


**Values** - Yes, you can use *null* as value in TreeMap:
```java
@Test
public void treeMapNullValueTest() {
    Map<String, String> treeMap = new TreeMap<>();
    treeMap.put("key", null);
    Assert.assertEquals(1, treeMap.size());
    Assert.assertTrue(treeMap.containsKey("key"));
    Assert.assertNull(treeMap.get("key"));
}
```

## Java 8 streams and Collectors.toMap()
Now let's assume that we want to use Java 8 streams to iterate over an existing map, make some changes and collect it back to Map. For this example, I'm going to filter entries where keys a lower than 4:
```java
@Test
public void hashMapCollectorsToMap() {
    Map<Integer, String> integersAndStrings = new HashMap<>(
            Map.of(
                1, "one",
                2, "two",
                3, "three",
                4, "four",
                5, "five"
    ));
    // set one of the values to null
    // everything is okay
    integersAndStrings.put(3, null);
    Assert.assertTrue(integersAndStrings.containsKey(3));
    Assert.assertNull(integersAndStrings.get(3));

    // here come the troubles
    Map<Integer, String> filteredMap = integersAndStrings.entrySet().stream()
            .filter(entry -> entry.getKey() < 4)
            .collect(Collectors.toMap(Map.Entry::getKey, Map.Entry::getValue));

    Assert.assertEquals(3 , filteredMap.size());
}
```
In this example we get *NullPointerException* when *Collection.toMap()* is called. But why? It's totally legit to have *null*-values in maps.  
The [javadoc of toMap()](https://docs.oracle.com/javase/8/docs/api/java/util/stream/Collectors.html#toMap-java.util.function.Function-java.util.function.Function-) explains that toMap() is based on Map.merge and the java doc of [Map.merge](https://docs.oracle.com/javase/8/docs/api/java/util/Map.html#merge-K-V-java.util.function.BiFunction-) says the following :
> @throws NullPointerException if the specified key is null and this map does not support null keys or the value or remappingFunction is null

Moreover, it's a [known bug](https://bugs.openjdk.java.net/browse/JDK-8148463) which is still unresolved.

So, we should use a workaround to avoid this exception. Here is one way to do so:
```java
@Test
public void hashMapCollectorsToMapFixed() {
    Map<Integer, String> integersAndStrings = new HashMap<>(
            Map.of(
                    1, "one",
                    2, "two",
                    3, "three",
                    4, "four",
                    5, "five"
            ));

    // set one of the values to null
    integersAndStrings.put(3, null);
    Assert.assertTrue(integersAndStrings.containsKey(3));
    Assert.assertNull(integersAndStrings.get(3));

    Map<Integer, String> filteredMap = integersAndStrings.entrySet().stream()
            .filter(entry -> entry.getKey() < 4)
            .collect(HashMap::new,
                    ((map, entry) -> map.put(entry.getKey(), entry.getValue())),
                    HashMap::putAll);

    Assert.assertEquals(3, filteredMap.size());
    Assert.assertTrue(filteredMap.containsKey(3));
    Assert.assertNull(filteredMap.get(3));
}
```
This way, there's no exceptions.

Have you seen other inconsistencies or strange behavior of Maps? Write about your experience in comments below :)

