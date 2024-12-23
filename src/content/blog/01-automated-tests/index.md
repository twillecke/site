---
title: "Automated Tests"
description: "What are automated tests and why should you implement them?"
date: "Dec 23 2024"
---
*This is the text version of the talk I gave at the last edition of [2024 Python Floripa](https://www.youtube.com/live/ECC7HjvzNXs?si=nWvqZaxo0cnJbXeT&t=2239).*

---

When we implement a program, procedure or class, we have expectations about its behaviour:

- In a sum function, we expect that when passing the numbers “3” and “4” as arguments, the result will be “7”.
- In a repository method for inserting a resource into a database, we expect it to be persisted, which would be confirmed by a query to the database.
- In these situations, how do we ensure that our solution works?

Often by testing manually via the terminal, with print(), interacting with the graphical interface, or with the - mysteriously forgotten - debugger.

For a simple component this may seem sufficient, but what about cases where there are several execution flows, exception throwing or complex state mutation?

We'll have to test every possible flow of our component with every modification of the source code!

The repetitive nature of manual testing leads us to error and frustration, concentrating our efforts on manual rather than creative activity.

In addition, testing gives the developer confidence that implementing new features or refactoring will not produce regressions:

> “Test-driven development is a way to manage fear while programming"- Kent Beck

Software development is many times a collective activity.

It is important to ensure that the feature we are developing does not produce side effects in other parts of the system.

As it is unfeasible for the developer to test the entire application, an alternative is to develop a continuous integration (CI) pipeline with automated test execution, delegating most of this effort to the machine.

### Testing behaviour

There are various approaches to implementing automated tests. We'll stick to the one proposed by Daniel-Terhorst and Chris Matts in Behaviour Driven Development (BDD).

As its name suggests, the proposal is to test the behaviour of components in an agnostic way - independent of implementation details -.

In this proposal, a test consists of three stages:

- Given: the input; or initial state of the application to test behaviour.
- When: the execution of the behaviour to be tested
- Then: the comparison between the expected behaviour and the behaviour produced.

In Python, we can use the Pytest framework to implement and execute tests. In the context of a sum method:
```python
def test_add():
    calc = Calculator()     # GIVEN a calculator instance
    result = calc.add(2, 2) # WHEN I call the add method with 2 and 2
    assert result == 4      # THEN the result should be 4
```

We can also test exceptions and error messages:
```python
def test_divide_by_zero():
    calc = Calculator()     # GIVEN a calculator instance
    try:
        calc.divide(5, 0)   # WHEN I divide a number by 0
    except ValueError as e:
        assert str(e) == 'Cannot divide by zero' # THEN an exception should be raised with the message 'Cannot divide by zero'
```

With a comprehensive set of tests, anyone can quickly test a system, with both failure and success flows covered.

Note that these are very granular examples. Units of behaviour are being tested in isolation.
However, the applications we develop have a much more complex cycle. The data is processed asynchronously by various integrated components; travels over the network, is serialized and deserialized.
This complexity makes relevant the...

### Integration Tests

Although there is no strict definition of integration testing, let's consider integration as the circumstance in which the application interacts with external systems, such as the file system, a web service or a database.

Let's imagine a movie catalogue application using TypeScript - the strongly typed variant of JavaScript - and the Jest testing framework.

It interacts with an SQL database to persist relevant data about movies.

A movie, the Movie entity, has the attributes: movie_id, create_at, title, director, rating.

All operations related to interaction with the database are implemented in the repository layer with the following methods:

- `save(Movie)`: inserts the attributes of a Movie entity into the movies table in the database.
- `getMovieById(movieId)`: returns all the attributes of a movie by movie_id.
- `deleteMovieById(movieId)`: removes all attributes related to a movie_id from the database.

```ts
export default class MovieRepositoryDatabase implements MovieRepository {
	constructor(readonly connection: DatabaseConnection) {}

	async save(movie: Movie) {
		await this.connection.query(
			"insert into movie_store.movies (movie_id, created_at, title, director, rating) values ($1, $2, $3, $4, $5)",
			[
				movie.getMovieId(),
				movie.getCreatedAt().toDateString(),
				movie.getMovietitle(),
				movie.getDirector(),
				movie.getRating(),
			],
		);
	}

	async getByMovieById(movieId: string) {
		const [movieData] = await this.connection.query(
			"SELECT movie_id, created_at, title, director, rating FROM movie_store.movies WHERE movie_id = $1",
			[movieId],
		);
		if (!movieData) return;
		return Movie.restore(
			movieData.movie_id,
			movieData.created_at,
			movieData.title,
			movieData.director,
			movieData.rating,
		);
	}

	async deleteByMovieId(movieId: string) {
		await this.connection.query(
			"delete from movie_store.movies where movie_id = $1",
			[movieId],
		);
	}
}
```
How could we implement a test for the operations of this repository?

Starting with the save(movie) method, following the “**Given**, **When**, **Then**” pattern:

1. **Given**, or the initial state of the application to test the behavior:

- SQL database named movie_store and movies table, with all the attributes of the Movie entity: movie_id, create_at, title, director, rating.
- Connection to the database, abstracted by the `DatabaseConnection` entity.
- Instance of `MovieRepository` passing the `DatabaseConnection` entity as an argument.
- Instance of the Movie entity with the attributes to be persisted in the database

```ts
const connection = new DatabaseConnection();
const movieRepository = new MovieRepositoryDatabase(connection);
const movie = new Movie("1", new Date(), "Inception", "Christopher Nolan", 5);
```

2. **When**, the behavior to be tested:
We invoke the save(Movie) method of the MovieRepository passing the Movie entity as an argument
```ts
await movieRepository.save(movie);
```

3. **Then**, the expected behavior:
We expect the movie to be persisted in the database, which would be confirmed by a query to the database.
```ts
expect(savedMovie).toEqual(movie);
```
Full implementation of the test:
```ts
test("deve salvar um filme na base de dados", async () => {
	const connection = new DatabaseConnection();
	const movieRepository = new MovieRepositoryDatabase(connection);
	const movie = new Movie("1", new Date(), "Inception", "Christopher Nolan", 5);
	await movieRepository.save(movie);
	const savedMovie = await movieRepository.getByMovieById("1");
	expect(savedMovie).toEqual(movie);
});
```

Compared to unit tests, integration tests follows a flow that is closer to the normal functioning of the application. It guarantees the basic functioning of our repository and the integration of the application with the database.

But it is a more expensive and complex test, because it requires a database execution and integration with other components. Even so, we can develop even more comprehensive tests: the...

### End-to-End Tests

In end-to-end testing, we test the application from the user interface.
Although not all applications have a graphical user interface (GUI), the predominance of web applications has led to the emergence of front-end testing frameworks such as pupeteer, playwright and cypress.

With them you can interact with DOM (document object model) elements, trigger events and check their status.
In an example with the cypress framework:

- **Given**: Accesses the address http://localhost:3000
- **When**:
  - Checks for the existence of input elements of a form with attributes input[name=“name” and input[email=“email” in the DOM
  - Fills in the input fields
  - Simulates the form submission click
- **Then**: Checks if an element containing “Thanks, John!” is visible to the user

```ts
describe('Cypress test example', () => {
  it('Should fill the form and check the result', () => {
    cy.visit('http://localhost:3000'); 
    cy.get('input[name="name"]').type('John Doe'); 
    cy.get('input[email="email"]').type('john.doe@example.com');
    cy.get('button[type="submit"]').click();
    cy.contains('Thanks, John!').should('be.visible');
  });
});
```

These are the most expensive tests in terms of the infrastructure required, the execution time and the layers of abstraction they cover. However, they are the closest to the complete flow of an application.

### The Test Pyramid
The test pyramid is a visual metaphor proposed by Mike Cohn to visualize the different layers of test types and how much to test in each one.
The higher the integration - as in end-to-end testing - the more expensive and slower the test execution. At the base of the pyramid, we have the performative unit tests.

The proposal is to concentrate the greatest number of tests at the lowest level, testing a wide variety of execution flows.And at the highest level, test only the main execution flows.

### Summary
- Automated tests save the developer from repetitive and error-prone efforts.
- They create a rapid feedback loop to preventively identify bugs; they provide security for refactoring and continuous integration.
- There are frameworks for testing even the graphical interfaces of web applications.
- They have different levels of granularity and integration, ideally leading us to implement a greater number of low-level tests.

### Resources
- Book: Kent Beck – Test Driven Development: by Example – 2002
- Pytest course – Harvard: https://library.cfa.harvard.edu/event/testing-python-code-pytest?delta=0
- TDD: Purposes and Practices – https://www.industriallogic.com/blog/tdd-purposes-and-practices/
- Test Pyramid: https://martinfowler.com/articles/practical-test-pyramid.html#TheImportanceOftestAutomation
- CI/CD: https://martinfowler.com/articles/continuousIntegration.html
