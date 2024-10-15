# Jpa-Specification

# What are JPA Specifications?
JPA Specifications are a way to create dynamic queries in a type-safe manner using the Criteria API. They allow you to define complex query conditions and easily combine them with logical operators (AND, OR).

# Key Components of JPA Specifications
```json
- Specification Interface: This is the core interface that allows you to define your custom query logic.
- CriteriaBuilder: A factory for constructing criteria queries, compound selections, expressions, predicates, and orderings.
- Predicate: Represents a condition in a query. You can combine predicates using logical operators.
```


# Code Explanation
Entity Classes
Employee Entity
```java
@Entity
@Table(name="EMPLOYEE")
public class Employee {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String name;
    private Integer salary;
    @DateTimeFormat(pattern="dd/MM/yyyy")
    private Date doj;
    private Subject skill;

    @OneToOne
    @JoinColumn(name = "address_id", referencedColumnName = "id")
    private Address address;
}
```


Represents an employee with attributes like id, name, salary, doj (date of joining), skill, and a reference to the Address entity.
# Address Entity

```java
@Entity
@Table(name ="ADDRESS")
public class Address implements Serializable {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private long id;
    private String city;
}
```

Represents an address associated with an employee.
# SpecificationInput Class

```java
public class SpecificationInput {
    private String columnName;
    private String value;
    private String sortColumn;
    private String sortOrder;
    private Integer pageNum;
    private Integer pageSize;
}
```
Contains the input parameters for specifying query conditions.
# SearchSpecification Class

```java
public class SearchSpecification {
    private String columnName;
    private String value;
    private String operation;
    private String joinTable;
}
```
Represents a search criterion that includes the column name, value, operation (e.g., EQUAL, LIKE), and the optional join table.
# RequestDTO Class

```java
public class RequestDTO {
    List<SearchSpecification> specificationList;
    String overallOperation;
}
```
Holds a list of search specifications and an overall operation (AND/OR).
# Repository Interface
```java
@Repository
public interface EmployeeRepo extends JpaRepository<Employee, Long>, JpaSpecificationExecutor<Employee> {
}
```
Extends JpaRepository for CRUD operations and JpaSpecificationExecutor for executing specifications.
# Service Class
```java
package com.filter.demo.service;

import com.filter.demo.model.Employee;
import com.filter.demo.model.RequestDTO;
import com.filter.demo.model.SearchSpecification;
import com.filter.demo.model.SpecificationInput;
import com.filter.demo.repo.EmployeeRepo;
import jakarta.annotation.PostConstruct;
import jakarta.annotation.PreDestroy;
import jakarta.persistence.criteria.Predicate;
import lombok.AllArgsConstructor;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.PageRequest;
import org.springframework.data.domain.Pageable;
import org.springframework.data.domain.Sort;
import org.springframework.data.jpa.domain.Specification;
import org.springframework.stereotype.Service;

import java.text.ParseException;
import java.text.SimpleDateFormat;
import java.util.ArrayList;
import java.util.Arrays;
import java.util.Date;
import java.util.List;

@Service
@AllArgsConstructor
public class EmployeeService {
    private final EmployeeRepo employeeRepo;

    @PostConstruct
    public void init() {
        System.out.println("Executing...");
    }

    @PreDestroy
    public void clear() {
        System.out.println("Closing...");
    }
          // name
    private Specification<Employee> getSpecification() {
        return (root, query, criteriaBuilder) -> criteriaBuilder.equal(root.get("name"), "ramu");
    }

    public List<Employee> getEmployeeByName() {
        Specification<Employee> specification = getSpecification();
        return employeeRepo.findAll(specification);
    }
         // Equal
    private Specification<Employee> getSpecification(SpecificationInput specificationInput) {
        return (root, query, criteriaBuilder) -> criteriaBuilder.equal(
                root.get(specificationInput.getColumnName()),
                specificationInput.getValue()
        );
    }

    public List<Employee> getEmployeeData(SpecificationInput specificationInput) {
        Specification<Employee> specification = getSpecification(specificationInput);
        return employeeRepo.findAll(specification);
    }
       //Between
    private Specification<Employee> getEmployeeSpecificationBetweenDates(SpecificationInput input) throws ParseException {
        String value = input.getValue();
        String[] values = value.split(",");
        SimpleDateFormat simpleDateFormat = new SimpleDateFormat("yyyy/MM/dd");
        Date startDt = simpleDateFormat.parse(values[0]);
        Date endDt = simpleDateFormat.parse(values[1]);
        return (root, query, criteriaBuilder) -> criteriaBuilder.between(
                root.get(input.getColumnName()), startDt, endDt
        );
    }
     
    public List<Employee> getEmployeesBetweenDates(SpecificationInput input) throws ParseException {
        Specification<Employee> empSpec = getEmployeeSpecificationBetweenDates(input);
        String sortColumn = input.getSortColumn();
        String sortOrder = input.getSortOrder();
        Integer pageNumber = input.getPageNum() >= 0 ? input.getPageNum() : 0;
        Integer pageSize = input.getPageSize() >= 0 ? input.getPageSize() : 2;

        boolean sortFlag = sortOrder.equalsIgnoreCase("ASC");

        Sort sort = Sort.by(sortFlag ? Sort.Direction.ASC : Sort.Direction.DESC, sortColumn);
        Pageable pageable = PageRequest.of(pageNumber, pageSize, sort);

        Page<Employee> pageEmployee = employeeRepo.findAll(empSpec, pageable);
        return pageEmployee.getContent();
    }
          // Like
    private Specification<Employee> getEmployeeSpecificationByLike(SpecificationInput input) {
        return (root, query, criteriaBuilder) -> criteriaBuilder.like(
                root.get(input.getColumnName()), "%" + input.getValue() + "%"
        );
    }

     
    public List<Employee> getEmployeeByLike(SpecificationInput specificationInput) {
        Specification<Employee> employeeSpecification = getEmployeeSpecificationByLike(specificationInput);
        return employeeRepo.findAll(employeeSpecification);
    }

      //greaterthanEqualto
    private Specification<Employee> getSpecificationByGreaterThan(SpecificationInput input) {
        return (root, query, criteriaBuilder) -> criteriaBuilder.greaterThanOrEqualTo(
                root.get(input.getColumnName()), input.getValue()
        );
    }

      // count,exists,delete
    public long getGreaterThan(SpecificationInput input) {
        Specification<Employee> specificationByGreaterThan = getSpecificationByGreaterThan(input);
        Long empCount = employeeRepo.count(specificationByGreaterThan);
        System.out.println("Emp count is :" + empCount);
        boolean emExists = employeeRepo.exists(specificationByGreaterThan);
        System.out.println("Emp Exists :" + emExists);
        long deleteStatus = 0;
        if (emExists) {
            deleteStatus = employeeRepo.delete(specificationByGreaterThan);
        }
        return deleteStatus;
    }

//In
    private Specification<Employee> getSpecificationByIn(SpecificationInput input) {
        String[] values = input.getValue().split(",");
        List<String> empList = Arrays.asList(values);
        return (root, query, criteriaBuilder) -> root.get(input.getColumnName()).in(empList);
    }

    public List<Employee> getEmployeeByIn(SpecificationInput specificationInput) {
        Specification<Employee> employeeSpecification = getSpecificationByIn(specificationInput);
        return employeeRepo.findAll(employeeSpecification);
    }

// combine search
    private Specification<Employee> getSpecificationForList(List<SearchSpecification> searchSpecificationList, String overallOp) {
        List<Predicate> predicateList = new ArrayList<>();
        return (root, query, criteriaBuilder) -> {
            for (SearchSpecification sf : searchSpecificationList) {
                String operation = sf.getOperation();
                switch (operation) {
                    case "EQUAL":
                        Predicate equal = criteriaBuilder.equal(root.get(sf.getColumnName()), sf.getValue());
                        predicateList.add(equal);
                        break;
                    case "GREATER_THAN":
                        Predicate greaterThan = criteriaBuilder.greaterThan(root.get(sf.getColumnName()), sf.getValue());
                        predicateList.add(greaterThan);
                        break;
                    case "GREATER_THAN_EQUAL":
                        Predicate greaterThanOrEqualTo = criteriaBuilder.greaterThanOrEqualTo(root.get(sf.getColumnName()), sf.getValue());
                        predicateList.add(greaterThanOrEqualTo);
                        break;
                    case "LESS_THAN":
                        Predicate lessThan = criteriaBuilder.lessThan(root.get(sf.getColumnName()), sf.getValue());
                        predicateList.add(lessThan);
                        break;
                    case "LESS_THAN_EQUAL":
                        Predicate lessThanOrEqualTo = criteriaBuilder.lessThanOrEqualTo(root.get(sf.getColumnName()), sf.getValue());
                        predicateList.add(lessThanOrEqualTo);
                        break;
                    case "LIKE":
                        Predicate like = criteriaBuilder.like(root.get(sf.getColumnName()), "%" + sf.getValue() + "%");
                        predicateList.add(like);
                        break;
                    case "IN":
                        String[] sp = sf.getValue().split(",");
                        Predicate in = root.get(sf.getColumnName()).in(List.of(sp));
                        predicateList.add(in);
                        break;
                    case "JOIN":
                        Predicate join = criteriaBuilder.equal(root.join(sf.getJoinTable()).get(sf.getColumnName()), sf.getValue());
                        predicateList.add(join);
                        break;
                }
            }
            if ("AND".equalsIgnoreCase(overallOp)) {
                return criteriaBuilder.and(predicateList.toArray(new Predicate[0]));
            } else {
                return criteriaBuilder.or(predicateList.toArray(new Predicate[0]));
            }
        };
    }
}

```
getEmployeeData: Uses a specification based on the input to retrieve employees matching the criteria.
getSpecification: Builds a specification based on the given column name and value.
# Dynamic Query Building
The EmployeeService class also contains methods to build specifications dynamically for various operations:

# - Between Dates:
# - Like Operation:             //These 3 are in service class
# - Combined Search:



# Controller Class
```java
package com.filter.demo.controller;

import com.filter.demo.model.Employee;
import com.filter.demo.model.RequestDTO;
import com.filter.demo.model.SearchSpecification;
import com.filter.demo.model.SpecificationInput;
import com.filter.demo.service.EmployeeService;
import lombok.AllArgsConstructor;
import org.springframework.web.bind.annotation.*;

import java.text.ParseException;
import java.util.List;

@RestController
@AllArgsConstructor
public class EmployeeController {
    private final EmployeeService employeeService;

    @GetMapping("/ByName")
    List<Employee> getByName() {
        return employeeService.getEmployeeByName();
    }

    @PostMapping("/ByEquals")
    List<Employee> ByEqual(@RequestBody SpecificationInput specificationInput) throws ParseException {
        return employeeService.getEmployeeData(specificationInput);
    }

    @PostMapping("/ByLike")
    List<Employee> ByLike(@RequestBody SpecificationInput specificationInput) {
        return employeeService.getEmployeeByLike(specificationInput);
    }

    @PostMapping("/ByIn")
    List<Employee> ByIn(@RequestBody SpecificationInput specificationInput) {
        return employeeService.getEmployeeByIn(specificationInput);
    }

    @PostMapping("/BetweenDates")
    List<Employee> getEmployeesBetweenDates(@RequestBody SpecificationInput input) throws ParseException {
        return employeeService.getEmployeesBetweenDates(input);
    }

    @PostMapping("/GreaterThan")
    long getGreaterThan(@RequestBody SpecificationInput input) {
        return employeeService.getGreaterThan(input);
    }

    @PostMapping("/CombinedSearch")
    List<Employee> getCombinedSearch(@RequestBody RequestDTO requestDTO) {
        List<SearchSpecification> searchSpecs = requestDTO.getSpecificationList();
        String overallOperation = requestDTO.getOverallOperation();
        return employeeService.getSpecificationForList(searchSpecs, overallOperation);
    }
}

```

Defines endpoints to retrieve employee data using the service methods based on various criteria.

# Notes Summary
```json
- Advantages: JPA Specifications provide a flexible and powerful way to build dynamic queries based on user input or application logic without writing complex JPQL or SQL.
- Combining Conditions: You can easily combine multiple specifications to create complex queries.
- Reusability: Specifications can be reused across different queries, promoting clean code and separation of concerns.
- These notes should help you understand JPA Specifications and how they can be effectively utilized in your Java Spring applications!
```
