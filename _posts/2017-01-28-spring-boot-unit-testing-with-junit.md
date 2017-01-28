---
layout:     post
title:      Writing Unit Tests for Rest Services with Spring Boot and JUnit
date:       2017-01-28 12:31:19
summary:    Setting up a basic REST Service with Spring Boot is a cake walk. We will go one step further and add great unit tests! 
categories: [Spring Boot, REST Service]
permalink:  /unit-testing-for-spring-boot-rest-services
---

This guide will help you write great unit tests for your Spring Boot Rest Service. We will use a simple code example creating couple of simple rest services. 
 
## You will learn
- How to create a Get REST Service for retrieving the courses for a student?
- How to write a unit test for Get REST Service?
- How to create a Post REST Service for registering a course for student?
- How to write a unit test for POST Service?

## Tools you will need
- Maven 3.0+ is your build tool
- Your favorite IDE. We use Eclipse.
- JDK 1.8+

## Creating REST Services Application with Spring Initializr

Creating a REST service with Spring Initializr is a cake walk. We will use Spring Web MVC as our web framework.  

Spring Initializr( http://start.spring.io/) is great tool to bootstrap your Spring Boot projects.

![Image](/images/Spring-Initializr-Web.png "Web, Actuator and Developer Tools")   

As shown in the image above, following steps have to be done

- Launch Spring Initializr and choose the following
  - Choose `com.in28minutes.springboot` as Group
  - Choose `student-services` as Artifact
  - Choose following dependencies
    - Web
    - Actuator
    - DevTools
- Click Generate Project.
- Import the project into Eclipse.
- If you want to understand all the files that are part of this project, you can go here.

## Implementing Business Service for your Application
All applications need data. Instead of talking to a real database, we will use an `ArrayList` - kind of an in-memory data store.

A student can take multiple courses. A course has an id, name, description and a list of steps you need to complete to finish the course. A student has an id, name, description and a list of courses he/she is currently registered for. We have StudentService exposing methods to 

- `public List<Student> retrieveAllStudents()` - Retrieve details for all students
- `public Student retrieveStudent(String studentId)` - Retrieve a specific student details
- `public List<Course> retrieveCourses(String studentId)` - Retrieve all courses a student is registered for
- `public Course retrieveCourse(String studentId, String courseId)` - Retrieve details of a specific course a student is registered for
- `public Course addCourse(String studentId, Course course)` - Add a course to an existing student

 Refer to these files at the bottom of the article for exact implementation of the Service `StudentService` and the model classes `Course` and `Student`. 

- src/main/java/com/in28minutes/springboot/model/Course.java
- src/main/java/com/in28minutes/springboot/model/Student.java
- src/main/java/com/in28minutes/springboot/service/StudentService.java


## Adding Couple of GET Rest Services

The Rest Service `StudentController` exposes couple of get services.

- `@Autowired private StudentService studentService` : We are using Spring Autowiring to wire the student service into the StudentController.
- `@GetMapping("/students/{studentId}/courses")`: Exposing a Get Service with studentId as a path variable 
- `@GetMapping("/students/{studentId}/courses/{courseId}")`: Exposing a Get Service for retrieving specific course of a student. 
- `@PathVariable String studentId`: Value of studentId from the uri will be mapped to this parameter.

```java
package com.in28minutes.springboot.controller;

import java.util.List;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RestController;

import com.in28minutes.springboot.model.Course;
import com.in28minutes.springboot.service.StudentService;

@RestController
public class StudentController {

	@Autowired
	private StudentService studentService;

	@GetMapping("/students/{studentId}/courses")
	public List<Course> retrieveCoursesForStudent(@PathVariable String studentId) {
		return studentService.retrieveCourses(studentId);
	}
	
	@GetMapping("/students/{studentId}/courses/{courseId}")
	public Course retrieveDetailsForCourse(@PathVariable String studentId,
			@PathVariable String courseId) {
		return studentService.retrieveCourse(studentId, courseId);
	}

}

```

## Executing the Get Service Using Postman
We will fire a request to http://localhost:8080/students/Student1/courses/Course1 to test the service. Response is as shown below.

```json
{
  "id": "Course1",
  "name": "Spring",
  "description": "10 Steps",
  "steps": [
    "Learn Maven",
    "Import Project",
    "First Example",
    "Second Example"
  ]
}
```

Below picture shows how we can execute this Get Service from Postman - my favorite tool to run rest services.
![Image](/images/ExecutingGetRestServiceUsingPostman.png "Executing Rest Service From Postman")   


## Unit Testing the Get Rest Service

When we are unit testing a rest service, we would want to launch only the specific controller and the related MVC Components. WebMvcTest annotation is used for unit testing Spring MVC application. This can be used when a test focuses only Spring MVC components. Using this annotation will disable full auto-configuration and only apply configuration relevant to MVC tests.

- `@RunWith(SpringRunner.class)` : SpringRunner is short hand for SpringJUnit4ClassRunner which extends BlockJUnit4ClassRunner providing the functionality to launch a Spring TestContext Framework.
- `@WebMvcTest(value = StudentController.class, secure = false)`: WebMvcTest annotation is used for unit testing Spring MVC application. This can be used when a test focuses only Spring MVC components. In this test, we want to launch only StudentController. All other controllers and mappings will not be launched when this unit test is executed. 
- `@Autowired private MockMvc mockMvc`: MockMvc is the main entry point for server-side Spring MVC test support. It allows us to execute requests against the test context.
- `@MockBean private StudentService studentService`: MockBean is used to add mocks to a Spring ApplicationContext. A mock of studentService is created and auto-wired into the StudentController.
- `Mockito.when(studentService.retrieveCourse(Mockito.anyString(),Mockito.anyString())).thenReturn(mockCourse)`: Mocking the method retrieveCourse to return the specific mockCourse when invoked.
- `MockMvcRequestBuilders.get("/students/Student1/courses/Course1").accept(MediaType.APPLICATION_JSON)`: Creating a Request builder to be able to execute a get request to uri "/students/Student1/courses/Course1" with accept header as "application/json"
- `mockMvc.perform(requestBuilder).andReturn()`: mockMvc is used to perform the request and return the response back.
- `JSONAssert.assertEquals(expected, result.getResponse().getContentAsString(), false)`: We are using org.skyscreamer.jsonassert.JSONAssert. This allows us to do partial asserts against a JSON String. We are passing strict as false since we do not want to check for all fields in the response.

```
package com.in28minutes.springboot.controller;

import java.util.Arrays;

import org.junit.Test;
import org.junit.runner.RunWith;
import org.mockito.Mockito;
import org.skyscreamer.jsonassert.JSONAssert;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
import org.springframework.boot.test.mock.mockito.MockBean;
import org.springframework.http.MediaType;
import org.springframework.test.context.junit4.SpringRunner;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.test.web.servlet.MvcResult;
import org.springframework.test.web.servlet.RequestBuilder;
import org.springframework.test.web.servlet.request.MockMvcRequestBuilders;

import com.in28minutes.springboot.model.Course;
import com.in28minutes.springboot.service.StudentService;

@RunWith(SpringRunner.class)
@WebMvcTest(value = StudentController.class, secure = false)
public class StudentControllerTest {

	@Autowired
	private MockMvc mockMvc;

	@MockBean
	private StudentService studentService;

	Course mockCourse = new Course("Course1", "Spring", "10 Steps",
			Arrays.asList("Learn Maven", "Import Project", "First Example",
					"Second Example"));

	String exampleCourseJson = "{\"name\":\"Spring\",\"description\":\"10 Steps\",\"steps\":[\"Learn Maven\",\"Import Project\",\"First Example\",\"Second Example\"]}";

	@Test
	public void retrieveDetailsForCourse() throws Exception {

		Mockito.when(
				studentService.retrieveCourse(Mockito.anyString(),
						Mockito.anyString())).thenReturn(mockCourse);

		RequestBuilder requestBuilder = MockMvcRequestBuilders.get(
				"/students/Student1/courses/Course1").accept(
				MediaType.APPLICATION_JSON);

		MvcResult result = mockMvc.perform(requestBuilder).andReturn();

		System.out.println(result.getResponse());
		String expected = "{id:Course1,name:Spring,description:10 Steps}";

		// {"id":"Course1","name":"Spring","description":"10 Steps, 25 Examples and 10K Students","steps":["Learn Maven","Import Project","First Example","Second Example"]}

		JSONAssert.assertEquals(expected, result.getResponse()
				.getContentAsString(), false);
	}

}

```

## Adding a POST Rest Service

A POST Service should return a status of created (201) when the resource creation is successful. 

`@PostMapping("/students/{studentId}/courses")`: Mapping a url for the POST Request
`@RequestBody Course newCourse`: Using Binding to bind the body of the request to Course object.
`ResponseEntity.created(location).build()`: Return a status of created. Also return the location of created resource as a Response Header.

```java
	@PostMapping("/students/{studentId}/courses")
	public ResponseEntity<Void> registerStudentForCourse(
			@PathVariable String studentId, @RequestBody Course newCourse) {

		Course course = studentService.addCourse(studentId, newCourse);

		if (course == null)
			return ResponseEntity.noContent().build();

		URI location = ServletUriComponentsBuilder.fromCurrentRequest().path(
				"/{id}").buildAndExpand(course.getId()).toUri();

		return ResponseEntity.created(location).build();
	}

```

## Executing a POST Rest Service

Example Request is shown below. It contains all the details to register a course to a student. 
```json
{
  "name": "Microservices",
  "description": "10 Steps",
  "steps": [
    "Learn How to Break Things Up",
    "Automate the hell out of everything",
    "Have fun"
  ]
}
```

Below picture shows how we can execute this Post Service from Postman - my favorite tool to run rest services. Make sure you go to the Body tab and select raw. Select JSON from the dropdown. Copy above request into body.

The URL we use is http://localhost:8080/students/Student1/courses.

![Image](/images/ExecutingPostRestServiceUsingPostman.png "Executing Post Rest Service From Postman")   

## Writing Unit Test for the POST Rest Service

In the unit test, we would want to post the request body to the url `/students/Student1/courses`. In the response, we check for HttpStatus of Created and that the location header contains the url of the created resource.

`MockMvcRequestBuilders.post("/students/Student1/courses").accept(MediaType.APPLICATION_JSON)`: Create a post request with an accept header for `application\json`
`content(exampleCourseJson).contentType(MediaType.APPLICATION_JSON)`: Use the specified content as body of the request and set content type header.
`assertEquals(HttpStatus.CREATED.value(), response.getStatus())`: Assert that the return status is CREATED.
`response.getHeader(HttpHeaders.LOCATION)`: Get the location from response header and later assert that it contains the URI of the created resource.

```java
	@Test
	public void createStudentCourse() throws Exception {
		Course mockCourse = new Course("1", "Smallest Number", "1",
				Arrays.asList("1", "2", "3", "4"));

		// studentService.addCourse to respond back with mockCourse
		Mockito.when(
				studentService.addCourse(Mockito.anyString(),
						Mockito.any(Course.class))).thenReturn(mockCourse);

		// Send course as body to /students/Student1/courses
		RequestBuilder requestBuilder = MockMvcRequestBuilders
				.post("/students/Student1/courses")
				.accept(MediaType.APPLICATION_JSON).content(exampleCourseJson)
				.contentType(MediaType.APPLICATION_JSON);

		MvcResult result = mockMvc.perform(requestBuilder).andReturn();

		MockHttpServletResponse response = result.getResponse();

		assertEquals(HttpStatus.CREATED.value(), response.getStatus());

		assertEquals("http://localhost/students/Student1/courses/1",
				response.getHeader(HttpHeaders.LOCATION));

	}

```

## Next Steps
- Understand more about Integration Testing
- Understand more about ****
- Understand more about ****
- Understand more about ****
- Understand more about ****
- Understand more about ****
