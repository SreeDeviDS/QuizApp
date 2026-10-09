# Quiz App (Microservices)

A quiz platform built as small Spring Boot services. One service manages the question bank, another builds and scores quizzes, a registry lets them find each other, and a gateway is the single entry point.

## Architecture

```
Client / Postman
      |
      v
 api-gateway (8082)  --->  service-registry (Eureka, 8761)
      |                          ^   ^
      |  routes by service name  |   |  register
      v                          |   |
 question-service (8080) <----- quiz-service (8081)
      |        (OpenFeign call)       |
      v                               v
  MySQL: companydb               MySQL: companydb2
```

| Service | Port | Role |
|---|---|---|
| service-registry | 8761 | Eureka server for service discovery |
| question-service | 8080 | CRUD for questions, random question selection, scoring |
| quiz-service | 8081 | Creates quizzes (stores question ids) and calls question-service via Feign |
| api-gateway | 8082 | Single entry point, routes requests using the service name |

Each service has its own database (database per service).

## Tech stack
Java, Spring Boot 4.1.1, Spring Cloud 2025.1.3 (Eureka, OpenFeign, Gateway), Spring Data JPA, MySQL, Lombok, Maven.

## What was built
- Four independent Spring Boot services: service registry, question-service, quiz-service and an API gateway
- **Service discovery** with Eureka: services register themselves and find each other by name
- **Inter-service communication** with OpenFeign: quiz-service fetches questions from question-service
- **API gateway** with discovery-based routing: one entry point (port 8082) for all services
- **Database per service**: separate MySQL databases for questions and quizzes
- **Question management (REST):** add, update, list all, list by category
- **Random quiz generation:** a native query picks random question ids for a category
- **Answer protection:** a `QuestionWrapper` response so quiz takers never receive the right answers
- **Scoring:** submitted answers are checked against the stored right answers
- Database password supplied through an environment variable, not stored in code
- All endpoints tested with Postman, directly and through the gateway

## Challenges solved
- **Build failure (`ExceptionInInitializerError` from Lombok):** caused by a JDK that was newer than Lombok supports. Fixed by switching the project SDK and Maven runner to a supported JDK and letting Spring Boot manage the Lombok version.
- **Feign client not found at startup:** fixed by adding `@EnableFeignClients`.
- **Gateway discovery properties not recognised:** the property names changed with the WebFlux gateway starter on Spring Cloud 2025.x (`spring.cloud.gateway.server.webflux...`).
- **405 errors in Postman:** endpoints that take a JSON body are POST (or PUT), so the wrong HTTP method was the cause.

## Prerequisites
- JDK 17 to 25. Do not use a newer JDK than your Lombok version supports (a JDK that is too new causes an `ExceptionInInitializerError` in the compiler).
- MySQL running locally on port 3306
- Maven (or the included `mvnw`)

## Setup

1. Create the two databases:
   ```sql
   CREATE DATABASE companydb;
   CREATE DATABASE companydb2;
   ```
2. Set the database password as an environment variable (never commit passwords):
   - Windows PowerShell: `$env:DB_PASSWORD="your_password"`
   - Linux/Mac: `export DB_PASSWORD=your_password`
3. Start the services **in this order**, each from its own folder with `./mvnw spring-boot:run`:
   1. `service-registry`
   2. `question-service`
   3. `quiz-service`
   4. `api-gateway`
4. Open `http://localhost:8761` and confirm `QUESTION-SERVICE`, `QUIZ-SERVICE` and `API-GATEWAY` are listed.

Tables are created by Hibernate on startup (`ddl-auto=update`). Add sample questions through the API, or insert rows after the `question` table exists:

```sql
USE companydb;
INSERT INTO question (category, difficultylevel, option1, option2, option3, option4, question_title, right_answer) VALUES
('Java', 'Easy', 'JVM', 'JDK', 'JRE', 'JIT', 'What does JVM stand for?', 'JVM'),
('Java', 'Easy', 'ArrayList', 'HashMap', 'HashSet', 'TreeMap', 'Which collection stores elements using key-value pairs?', 'HashMap'),
('SQL', 'Medium', 'INNER JOIN', 'LEFT JOIN', 'RIGHT JOIN', 'FULL JOIN', 'Which JOIN returns all rows from the left table?', 'LEFT JOIN');
```

## API endpoints

Call the services directly, or through the gateway at `http://localhost:8082/<service-name>/...` (for example `http://localhost:8082/question-service/question/allQuestions`).

### question-service (`/question`)
| Method | Path | Description |
|---|---|---|
| GET | `/allQuestions` | All questions (includes right answers) |
| GET | `/category/{category}` | Questions of one category |
| POST | `/add` | Add a question (JSON body) |
| PUT | `/update/{id}` | Update a question (JSON body) |
| GET | `/generate?categoryName=Java&numQuestions=3` | Random question ids for a quiz |
| POST | `/getQuestions` | Body `[1, 4, 7]`: questions without right answers |
| POST | `/getScore` | Body `[{"id":1,"response":"JVM"}]`: returns the score |

### quiz-service (`/quiz`)
| Method | Path | Description |
|---|---|---|
| POST | `/create` | Body `{"categoryName":"Java","numQuestions":3,"title":"Java Quiz 1"}` |
| POST | `/get/{id}` | Questions of a quiz (no right answers) |
| POST | `/submit/{id}` | Submit answers, returns the score |

Sample request body for adding a question:
```json
{
  "questionTitle": "Which keyword prevents a method from being overridden?",
  "option1": "static",
  "option2": "final",
  "option3": "abstract",
  "option4": "private",
  "rightAnswer": "final",
  "difficultylevel": "Easy",
  "category": "Java"
}
```

## Design notes
- **Service discovery:** services register with Eureka and find each other by name, not by hard-coded address.
- **Inter-service calls:** quiz-service uses OpenFeign to call question-service.
- **Hidden answers:** a `QuestionWrapper` class is returned to quiz takers so the right answer is never sent to the client.
- **Quiz stores ids only:** quiz-service keeps question ids, and fetches questions from question-service when needed.

## Known limitations (high priority)
- **Scoring:** `getScore` counts duplicate ids more than once, and does not check that the submitted answers belong to the quiz.
- **Error handling:** an unknown id returns a 500; global exception handling (`@ControllerAdvice`) and input validation are still to be added.
- **Resilience:** if question-service is down, quiz-service fails; a circuit breaker (Resilience4j) with a fallback is the planned fix.
