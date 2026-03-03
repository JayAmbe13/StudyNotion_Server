# AGENTS.md

This file provides guidance to WARP (warp.dev) when working with code in this repository.

## Project Overview

StudyNotion backend — a Node.js/Express REST API for an ed-tech course platform. Uses MongoDB (Mongoose), Cloudinary for media uploads, Razorpay for payments, and Nodemailer for transactional emails. CommonJS modules throughout (`require`/`module.exports`).

## Commands

- **Start (production):** `npm start` — runs `node index.js`
- **Start (development):** `npm run dev` — runs `nodemon index.js` (auto-restarts on file changes)
- **No test framework, linter, or type checker is configured.**

## Environment

All configuration is via `.env` file in the project root. Required variables:

- `PORT`, `MONGODB_URL`, `JWT_SECRET`
- `MAIL_HOST`, `MAIL_USER`, `MAIL_PASS` (SMTP for Nodemailer)
- `RAZORPAY_KEY`, `RAZORPAY_SECRET`
- `CLOUD_NAME`, `API_KEY`, `API_SECRET` (Cloudinary)
- `FOLDER_NAME` (Cloudinary upload folder)

The server defaults to port 4000 and expects a frontend at `http://localhost:3000` (CORS origin).

## Architecture

### Entry Point & Middleware Stack (`index.js`)

Express app with: `express.json()`, `cookie-parser`, CORS (localhost:3000), `express-fileupload` (temp files to `/tmp`). Connects to MongoDB and Cloudinary on startup, then mounts four route groups.

### API Route Prefixes

- `/api/v1/auth` — signup, login, OTP send, password change/reset (`routes/User.js`)
- `/api/v1/profile` — user profile CRUD, display picture upload (`routes/Profile.js`)
- `/api/v1/course` — course/section/subsection CRUD, categories, ratings & reviews (`routes/Course.js`)
- `/api/v1/payment` — Razorpay payment capture and webhook signature verification (`routes/Payments.js`)

### Authentication & Authorization (`middlewares/auth.js`)

JWT-based. The `auth` middleware extracts tokens from (in order): `req.cookies.token`, `req.body.token`, or the `Authorisation` header (note: British spelling, not `Authorization`). Three role-check middlewares: `isStudent`, `isInstructor`, `isAdmin` — all check `req.user.accountType`.

User account types are: `"Student"`, `"Instructor"`, `"Admin"`.

### Data Model Relationships

- **User** → refs `Profile` (via `additionalDetails`), `Course[]` (enrolled/created courses), `CourseProgress[]`
- **Course** → refs `User` (instructor), `Section[]` (courseContent), `Category`, `RatingAndReview[]`, `User[]` (studentsEnrolled)
- **Section** → refs `SubSection[]`
- **SubSection** — stores `videoUrl`, `timeDuration`, `description`
- **Category** → refs `Course[]`
- **OTP** — has a Mongoose `pre("save")` hook that automatically sends a verification email on creation. Documents auto-expire after 5 minutes via TTL index.

Note: The User model is registered as `"user"` (lowercase) while Course is `"Course"` (capitalized). References must match these exact model names.

### Utilities

- `utils/mailSender.js` — generic Nodemailer transport wrapper (`email`, `title`, `body` → sends HTML email)
- `utils/imageUploader.js` — Cloudinary upload wrapper; `resource_type: "auto"` handles both images and videos. Used for course thumbnails, subsection videos, and profile pictures.

### Email Templates (`mail/templates/`)

Functions returning HTML strings for: OTP verification, course enrollment confirmation, password update notification. These are passed as the `body` argument to `mailSender`.

### Config (`config/`)

- `database.js` — Mongoose connection (exits process on failure)
- `cloudinary.js` — Cloudinary SDK configuration
- `razorpay.js` — Razorpay instance (exports `instance`)

## Conventions

- All API responses follow the shape `{ success: boolean, message: string, data?: any }`.
- Controllers are organized one-per-domain-entity in `controllers/` and exported as named functions.
- Routes apply auth middleware first, then role middleware, then the controller.
- File uploads arrive via `req.files` (from `express-fileupload`) and are uploaded to Cloudinary using the shared `uploadImageToCloudinary` utility.
- Passwords are hashed with bcrypt (salt rounds: 10).
- JWT tokens expire after 24 hours; cookies expire after 3 days.
