# EV Charging Station Management System

## 1. Project Overview
The EV Charging Station Management System is a backend application built to manage electric vehicle charging stations. It allows administrators to monitor stations, manage charging ports, and handle subscriptions. Drivers can use the system to find available stations in real-time, simulate charging sessions, book ports in advance, and pay for services using an integrated wallet system through ZaloPay.

## 2. Technologies Used
- Java
- Spring Boot (Spring Web, Spring Data JPA)
- Spring Security & JWT
- MySQL
- Swagger UI / OpenAPI 3
- ZaloPay API
- Cloudinary
- Spring Mail

## 3. Installation & Setup
Follow these steps to configure and run the project locally.

**Prerequisites:**
- Java 21 (or compatible minimum)
- Maven
- MySQL Server

**Step-by-step Setup:**
1. **Clone the repository:**
   ```bash
   git clone https://github.com/khoahtd/ev-charging-station-management-system.git
   cd ev-charging-station-management-system
   ```

2. **Database Initialization:**
   - Create a MySQL database (e.g., `railway`).
   - Run the initial SQL scripts or run any migration files located in the `sql/` directory to generate tables and seed data if necessary. Alternatively, you can rely on Hibernate's `ddl-auto: update`.

3. **Configure Environment Variables:**
   For security, sensitive keys are not committed. Set the following environment variables on your machine (or replace them in `src/main/resources/application.yaml`):
   - `DB_URL` (e.g., `jdbc:mysql://localhost:3306/railway`), `DB_USERNAME`, `DB_PASSWORD`
   - `JWT_SECRET` (A 256-bit secure secret key)
   - `MAIL_USERNAME`, `MAIL_PASSWORD` (App password for email forwarding)
   - `ZALOPAY_APP_ID`, `ZALOPAY_KEY1`, `ZALOPAY_KEY2` (ZaloPay Sandbox keys)
   - `CLOUDINARY_CLOUD_NAME`, `CLOUDINARY_API_KEY`, `CLOUDINARY_API_SECRET`
   - `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET` (For OAuth2 Google login)

4. **Run the Application:**
   ```bash
   ./mvnw spring-boot:run
   ```
   The backend will start on port `8080` with the context path `/evchargingstation`.

## 4. API Documentation
We maintain an extensive, modular API documentation detailing all endpoints, authentication procedures, and expected request/response formats.

**[View the Full API Documentation](./docs/API_DOCUMENTATION.md)**

### Interactive Testing with Swagger UI
Once you have the application running locally, you can view and test the API visually via Swagger UI:

**URL:** `http://localhost:8080/evchargingstation/swagger-ui.html`
