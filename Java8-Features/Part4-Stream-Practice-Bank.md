# Part 4 — Stream Interview Practice Bank

> A worked bank of the exact Stream questions that show up in real interviews (Amazon/Flipkart-style), all built against one running `Employee` dataset — min/max lookups, department/company/location aggregations, the N-th-highest-value pattern, and `flatMap` over nested collections. Each one states *why* it's asked, not just the code. Interview Q&A at the end.

## The Dataset Used Throughout

```java
public class Employee {
    int empId;
    String name;
    String dept;
    String company;
    String location;
    double salary;
    List<String> projects;
    // constructor, getters/setters omitted

    public static List<Employee> employeeData() {
        return List.of(
            new Employee(1, "Anurag", "IT", "TCS", "Bangalore", 65000, List.of("Alpha", "Beta")),
            new Employee(2, "Ravi", "HR", "Infosys", "Pune", 48000, List.of("Gamma")),
            new Employee(3, "Aarti", "Finance", "TCS", "Noida", 72000, List.of("Beta", "Delta")),
            new Employee(4, "John", "IT", "Wipro", "Bangalore", 58000, List.of("Alpha", "Delta"))
        );
    }
}
List<Employee> employees = Employee.employeeData();
```

## Min / Max Lookups

```java
Employee minSalaryEmp = employees.stream()
    .min(Comparator.comparingDouble(Employee::getSalary))
    .orElse(null);

Employee maxSalaryEmp = employees.stream()
    .max(Comparator.comparingDouble(Employee::getSalary))
    .orElse(null);

String topCompany = employees.stream()
    .max(Comparator.comparingDouble(Employee::getSalary))
    .map(Employee::getCompany)
    .orElse("No data");
```
**Why it's asked:** tests whether you reach for `.min()`/`.max()` with a `Comparator` directly, instead of manually sorting the whole list and grabbing the first/last element (correct, but wasteful — sorting is `O(n log n)`, `min`/`max` is a single `O(n)` pass).

## Department / Company / Location Aggregations

```java
// Count per department
Map<String, Long> countByDept = employees.stream()
    .collect(Collectors.groupingBy(Employee::getDept, Collectors.counting()));

// Sum of salaries per department
Map<String, Double> sumByDept = employees.stream()
    .collect(Collectors.groupingBy(Employee::getDept, Collectors.summingDouble(Employee::getSalary)));

// Average salary per company, sorted highest to lowest
employees.stream()
    .collect(Collectors.groupingBy(Employee::getCompany, Collectors.averagingDouble(Employee::getSalary)))
    .entrySet().stream()
    .sorted(Map.Entry.comparingByValue(Comparator.reverseOrder()))
    .forEach(entry -> System.out.println(entry.getKey() + ": " + entry.getValue()));

// Highest-paid employee PER location
Map<String, Optional<Employee>> topEarnerByLocation = employees.stream()
    .collect(Collectors.groupingBy(Employee::getLocation, Collectors.maxBy(Comparator.comparingDouble(Employee::getSalary))));

// Which DEPARTMENT has the single highest-paid employee overall (a two-step aggregation)
employees.stream()
    .collect(Collectors.groupingBy(Employee::getDept, Collectors.maxBy(Comparator.comparingDouble(Employee::getSalary))))
    .entrySet().stream()
    .max(Comparator.comparingDouble(entry -> entry.getValue().get().getSalary()))
    .ifPresent(System.out::println);
```
**Why the last one is asked specifically:** it's a **two-level aggregation** — first `groupingBy` + `maxBy` to find each department's top earner, *then* a second stream pass over that resulting map to find which department's top earner is the overall highest. Interviewers use this to see whether you can chain aggregations rather than trying to do it all in one pass (which isn't cleanly possible here, since you need the per-department max *before* you can compare across departments).

> ⚠️ **Pitfall — sorting a `Map` by value:** a `Map` has no inherent order, so "sort by value" always means converting `entrySet()` to a stream, sorting *that*, and consuming the result as an ordered sequence (`forEach`, or collecting into a `LinkedHashMap` if you need an actual ordered map back) — you cannot sort a `HashMap` in place. `Map.Entry.comparingByValue(Comparator.reverseOrder())` is the idiomatic one-liner for descending-by-value.

## Grouping by One Field, Collecting a Different Field

```java
Map<String, List<String>> namesByDept = employees.stream()
    .collect(Collectors.groupingBy(Employee::getDept, Collectors.mapping(Employee::getName, Collectors.toList())));
// {IT=[Anurag, John], HR=[Ravi], Finance=[Aarti]}
```
**Why it's asked:** to check you know `Collectors.mapping()` as the downstream collector for "group by X, but I only want field Y from each element" — see Part 1's Collectors section for the full explanation of why a plain `groupingBy` alone can't do this.

## `flatMap` Over Nested Collections — Distinct Projects Across All Employees

```java
List<String> allDistinctProjects = employees.stream()
    .flatMap(e -> e.getProjects().stream())
    .distinct()
    .collect(Collectors.toList());
// [Alpha, Beta, Gamma, Delta]
```
**Why it's asked:** this is the canonical real-world `flatMap` question — "get me one flat, deduplicated list of X across all the Y's" whenever each Y itself holds a collection of X. See Part 1's full `flatMap` deep-dive for three more variations of this same underlying shape.

## Partitioning by a Threshold

```java
Map<Boolean, List<Employee>> partitioned = employees.stream()
    .collect(Collectors.partitioningBy(e -> e.getSalary() > 50000));
System.out.println("Above 50k: " + partitioned.get(true));
System.out.println("At or below 50k: " + partitioned.get(false));
```
**Why it's asked:** tests whether you reach for `partitioningBy` (guaranteed exactly two groups) instead of `groupingBy` with a boolean classifier — see Part 1 for the precise distinction between the two.

## The N-th Highest Value Pattern

```java
OptionalDouble thirdHighestSalary = employees.stream()
    .mapToDouble(Employee::getSalary)
    .distinct()     // remove duplicate salary values FIRST -- otherwise ties corrupt the "Nth highest" notion
    .sorted()        // ascending
    .skip(2)         // skip the two lowest of what remains -- 0-indexed, so skip(2) leaves the 3rd-lowest onward
    .findFirst();    // ascending + skip(n-1) = the n-th lowest; for n-th HIGHEST, reverse the sort direction or skip from the top
```
**Why this exact combination matters:** `distinct()` **must** come before `sorted().skip(n)` — if two employees share the same salary, skipping by raw position (without deduplicating first) would give you a wrong or duplicate-driven answer for "the 3rd *highest distinct* salary." This four-operation chain (`distinct` → `sorted` → `skip` → `findFirst`) is a genuinely common composed pattern worth having memorized as a unit, not derived from scratch under interview pressure.

> ⚠️ **Pitfall — ascending vs descending direction:** `sorted()` alone sorts ascending, so `skip(n-1).findFirst()` after a plain ascending sort gives you the **n-th lowest**, not highest. For the n-th **highest**, either sort descending first (`sorted(Comparator.reverseOrder())` on a boxed stream, since `DoubleStream.sorted()` has no reverse-order overload — you may need `.boxed().sorted(Comparator.reverseOrder())`), or compute `count() - n` and adjust the skip amount against an ascending sort. Getting the sort direction backwards here is an easy, common slip under interview time pressure.

---

## Interview Q&A

**Q: How would you find the employee with the highest salary, and then the company they work for?**
`employees.stream().max(Comparator.comparingDouble(Employee::getSalary)).map(Employee::getCompany).orElse(...)` — find the max employee first via a `Comparator`-driven `.max()`, then `.map()` the resulting `Optional<Employee>` down to just the company field.

**Q: How do you find which department has the single highest-paid employee, given salaries vary within each department?**
Two-step aggregation: first `groupingBy(dept, maxBy(salary comparator))` to get each department's top earner, then stream over that resulting map's entries and `max()` again by the nested employee's salary to find the overall top department. This can't be done in one `groupingBy` pass because you need every department's max computed before you can compare across departments.

**Q: You need names of employees per department, not the full employee objects — how?**
`Collectors.groupingBy(Employee::getDept, Collectors.mapping(Employee::getName, Collectors.toList()))` — `Collectors.mapping()` is the downstream collector specifically for "group by one field, collect a different extracted field."

**Q: Walk through finding the 3rd-highest distinct salary in a stream of employees.**
`mapToDouble(Employee::getSalary).distinct().sorted().skip(n).findFirst()` for the n-th *lowest* distinct value — `distinct()` must run before `sorted()`/`skip()` so that duplicate salaries don't corrupt the position count. For the n-th *highest*, either sort descending or compute the skip amount relative to the total count, since a plain ascending `sorted()` gives you the lowest end by default.

**Q: Why can't you sort a `Map` directly by its values?**
A `Map` has no inherent ordering contract — "sorting by value" always means streaming `entrySet()`, sorting that stream via `Map.Entry.comparingByValue(...)`, then consuming the sorted sequence (`forEach`, or re-collecting into a `LinkedHashMap` if an actual ordered map is needed back) — you cannot sort a `HashMap` (or most `Map` implementations) in place.
