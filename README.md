README.md
# Online Job Portal
Java Web Project using JSP, Servlet, JDBC, MySQL


## Features
- User Login & Registration
- Job Posting
- Job Viewing
- Database Connectivity


## How to Run
1. Import project in Eclipse
2. Configure Tomcat Server
3. Import database jobportal.sql
4. Run on server
---

// ---------- DB_SCHEMA.sql ----------
-- Run this in your MySQL / PostgreSQL (adjust types) to create tables

-- For MySQL
CREATE DATABASE IF NOT EXISTS job_portal;
USE job_portal;

CREATE TABLE IF NOT EXISTS users (
    user_id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    password VARCHAR(255) NOT NULL,
    role ENUM('JOB_SEEKER','EMPLOYER') NOT NULL,
    phone VARCHAR(20)
);

CREATE TABLE IF NOT EXISTS jobs (
    job_id INT AUTO_INCREMENT PRIMARY KEY,
    employer_id INT NOT NULL,
    title VARCHAR(150) NOT NULL,
    description TEXT,
    location VARCHAR(100),
    salary VARCHAR(50),
    posted_date DATE DEFAULT (CURRENT_DATE()),
    FOREIGN KEY (employer_id) REFERENCES users(user_id) ON DELETE CASCADE
);

-- ---------- DBConnection.java ----------
package util;

import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.SQLException;

public class DBConnection {
    private static final String URL = "jdbc:mysql://localhost:3306/job_portal"; // modify
    private static final String USER = "root"; // modify
    private static final String PASS = "password"; // modify

    static {
        try {
            Class.forName("com.mysql.cj.jdbc.Driver");
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        }
    }

    public static Connection getConnection() throws SQLException {
        return DriverManager.getConnection(URL, USER, PASS);
    }
}

// ---------- models/User.java ----------
package models;

public class User {
    private int userId;
    private String name;
    private String email;
    private String password;
    private String role; // JOB_SEEKER or EMPLOYER
    private String phone;

    public User() {}

    public User(int userId, String name, String email, String password, String role, String phone) {
        this.userId = userId;
        this.name = name;
        this.email = email;
        this.password = password;
        this.role = role;
        this.phone = phone;
    }

    // getters and setters
    public int getUserId() { return userId; }
    public void setUserId(int userId) { this.userId = userId; }
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    public String getEmail() { return email; }
    public void setEmail(String email) { this.email = email; }
    public String getPassword() { return password; }
    public void setPassword(String password) { this.password = password; }
    public String getRole() { return role; }
    public void setRole(String role) { this.role = role; }
    public String getPhone() { return phone; }
    public void setPhone(String phone) { this.phone = phone; }
}

// ---------- models/Job.java ----------
package models;

import java.sql.Date;

public class Job {
    private int jobId;
    private int employerId;
    private String title;
    private String description;
    private String location;
    private String salary;
    private Date postedDate;

    public Job() {}

    // getters and setters
    public int getJobId() { return jobId; }
    public void setJobId(int jobId) { this.jobId = jobId; }
    public int getEmployerId() { return employerId; }
    public void setEmployerId(int employerId) { this.employerId = employerId; }
    public String getTitle() { return title; }
    public void setTitle(String title) { this.title = title; }
    public String getDescription() { return description; }
    public void setDescription(String description) { this.description = description; }
    public String getLocation() { return location; }
    public void setLocation(String location) { this.location = location; }
    public String getSalary() { return salary; }
    public void setSalary(String salary) { this.salary = salary; }
    public Date getPostedDate() { return postedDate; }
    public void setPostedDate(Date postedDate) { this.postedDate = postedDate; }
}

// ---------- dao/UserDAO.java ----------
package dao;

import models.User;
import util.DBConnection;

import java.sql.*;
import java.util.Optional;

public class UserDAO {

    public static boolean register(User u) {
        String sql = "INSERT INTO users(name,email,password,role,phone) VALUES(?,?,?,?,?)";
        try (Connection con = DBConnection.getConnection(); PreparedStatement ps = con.prepareStatement(sql, Statement.RETURN_GENERATED_KEYS)) {
            ps.setString(1, u.getName());
            ps.setString(2, u.getEmail());
            ps.setString(3, u.getPassword()); // NOTE: store hashed password in production
            ps.setString(4, u.getRole());
            ps.setString(5, u.getPhone());
            int rows = ps.executeUpdate();
            if (rows > 0) {
                try (ResultSet rs = ps.getGeneratedKeys()) {
                    if (rs.next()) u.setUserId(rs.getInt(1));
                }
                return true;
            }
        } catch (SQLException e) {
            e.printStackTrace();
        }
        return false;
    }

    public static Optional<User> login(String email, String password) {
        String sql = "SELECT * FROM users WHERE email = ? AND password = ?";
        try (Connection con = DBConnection.getConnection(); PreparedStatement ps = con.prepareStatement(sql)) {
            ps.setString(1, email);
            ps.setString(2, password);
            try (ResultSet rs = ps.executeQuery()) {
                if (rs.next()) {
                    User u = new User();
                    u.setUserId(rs.getInt("user_id"));
                    u.setName(rs.getString("name"));
                    u.setEmail(rs.getString("email"));
                    u.setRole(rs.getString("role"));
                    u.setPhone(rs.getString("phone"));
                    return Optional.of(u);
                }
            }
        } catch (SQLException e) {
            e.printStackTrace();
        }
        return Optional.empty();
    }
}

// ---------- dao/JobDAO.java ----------
package dao;

import models.Job;
import util.DBConnection;

import java.sql.*;
import java.util.ArrayList;
import java.util.List;

public class JobDAO {

    public static boolean postJob(Job job) {
        String sql = "INSERT INTO jobs(employer_id,title,description,location,salary) VALUES(?,?,?,?,?)";
        try (Connection con = DBConnection.getConnection(); PreparedStatement ps = con.prepareStatement(sql, Statement.RETURN_GENERATED_KEYS)) {
            ps.setInt(1, job.getEmployerId());
            ps.setString(2, job.getTitle());
            ps.setString(3, job.getDescription());
            ps.setString(4, job.getLocation());
            ps.setString(5, job.getSalary());
            int r = ps.executeUpdate();
            if (r>0) {
                try (ResultSet rs = ps.getGeneratedKeys()) { if (rs.next()) job.setJobId(rs.getInt(1)); }
                return true;
            }
        } catch (SQLException e) { e.printStackTrace(); }
        return false;
    }

    public static List<Job> listAllJobs() {
        List<Job> list = new ArrayList<>();
        String sql = "SELECT * FROM jobs";
        try (Connection con = DBConnection.getConnection(); PreparedStatement ps = con.prepareStatement(sql); ResultSet rs = ps.executeQuery()) {
            while (rs.next()) {
                Job j = new Job();
                j.setJobId(rs.getInt("job_id"));
                j.setEmployerId(rs.getInt("employer_id"));
                j.setTitle(rs.getString("title"));
                j.setDescription(rs.getString("description"));
                j.setLocation(rs.getString("location"));
                j.setSalary(rs.getString("salary"));
                j.setPostedDate(rs.getDate("posted_date"));
                list.add(j);
            }
        } catch (SQLException e) { e.printStackTrace(); }
        return list;
    }

    public static List<Job> listJobsByEmployer(int employerId) {
        List<Job> list = new ArrayList<>();
        String sql = "SELECT * FROM jobs WHERE employer_id = ?";
        try (Connection con = DBConnection.getConnection(); PreparedStatement ps = con.prepareStatement(sql)) {
            ps.setInt(1, employerId);
            try (ResultSet rs = ps.executeQuery()) {
                while (rs.next()) {
                    Job j = new Job();
                    j.setJobId(rs.getInt("job_id"));
                    j.setEmployerId(rs.getInt("employer_id"));
                    j.setTitle(rs.getString("title"));
                    j.setDescription(rs.getString("description"));
                    j.setLocation(rs.getString("location"));
                    j.setSalary(rs.getString("salary"));
                    j.setPostedDate(rs.getDate("posted_date"));
                    list.add(j);
                }
            }
        } catch (SQLException e) { e.printStackTrace(); }
        return list;
    }
}

// ---------- Main.java ----------
import models.User;
import models.Job;
import dao.UserDAO;
import dao.JobDAO;

import java.util.List;
import java.util.Optional;
import java.util.Scanner;

public class Main {
    private static Scanner sc = new Scanner(System.in);
    private static User currentUser = null;

    public static void main(String[] args) {
        while (true) {
            if (currentUser == null) showWelcomeMenu();
            else showUserMenu();
        }
    }

    private static void showWelcomeMenu() {
        System.out.println("--- Online Job Portal ---");
        System.out.println("1. Register\n2. Login\n3. List Jobs\n0. Exit");
        System.out.print("Choose: ");
        String c = sc.nextLine();
        switch (c) {
            case "1": register(); break;
            case "2": login(); break;
            case "3": listJobs(); break;
            case "0": System.exit(0);
            default: System.out.println("Invalid");
        }
    }

    private static void showUserMenu() {
        System.out.println("\nLogged in as: " + currentUser.getName() + " (" + currentUser.getRole() + ")");
        if ("EMPLOYER".equals(currentUser.getRole())) {
            System.out.println("1. Post Job\n2. My Jobs\n3. Logout");
            System.out.print("Choose: ");
            String c = sc.nextLine();
            switch (c) {
                case "1": postJob(); break;
                case "2": myJobs(); break;
                case "3": logout(); break;
                default: System.out.println("Invalid");
            }
        } else {
            System.out.println("1. List Jobs\n2. Logout");
            System.out.print("Choose: ");
            String c = sc.nextLine();
            switch (c) {
                case "1": listJobs(); break;
                case "2": logout(); break;
                default: System.out.println("Invalid");
            }
        }
    }

    private static void register() {
        User u = new User();
        System.out.print("Name: "); u.setName(sc.nextLine());
        System.out.print("Email: "); u.setEmail(sc.nextLine());
        System.out.print("Password: "); u.setPassword(sc.nextLine());
        System.out.print("Role (JOB_SEEKER/EMPLOYER): "); u.setRole(sc.nextLine().toUpperCase());
        System.out.print("Phone: "); u.setPhone(sc.nextLine());
        boolean ok = UserDAO.register(u);
        System.out.println(ok ? "Registered successfully." : "Registration failed.");
    }

    private static void login() {
        System.out.print("Email: ");
        String email = sc.nextLine();
        System.out.print("Password: ");
        String pass = sc.nextLine();
        Optional<User> ou = UserDAO.login(email, pass);
        if (ou.isPresent()) {
            currentUser = ou.get();
            System.out.println("Welcome, " + currentUser.getName());
        } else System.out.println("Invalid credentials.");
    }

    private static void postJob() {
        Job j = new Job();
        j.setEmployerId(currentUser.getUserId());
        System.out.print("Job title: "); j.setTitle(sc.nextLine());
        System.out.print("Description: "); j.setDescription(sc.nextLine());
        System.out.print("Location: "); j.setLocation(sc.nextLine());
        System.out.print("Salary: "); j.setSalary(sc.nextLine());
        boolean ok = JobDAO.postJob(j);
        System.out.println(ok ? "Job posted." : "Failed to post job.");
    }

    private static void listJobs() {
        List<Job> list = JobDAO.listAllJobs();
        System.out.println("\n--- Jobs ---");
        for (Job j : list) {
            System.out.println("ID: " + j.getJobId() + " | Title: " + j.getTitle() + " | Loc: " + j.getLocation() + " | Salary: " + j.getSalary());
            System.out.println("Desc: " + (j.getDescription()==null?"-":j.getDescription()));
            System.out.println("Posted: " + j.getPostedDate());
            System.out.println("---------------------");
        }
    }

    private static void myJobs() {
        List<Job> list = JobDAO.listJobsByEmployer(currentUser.getUserId());
        System.out.println("\n--- My Jobs ---");
        for (Job j : list) System.out.println(j.getJobId() + ": " + j.getTitle() + " (" + j.getLocation() + ")");
    }

    private static void logout() { currentUser = null; System.out.println("Logged out."); }
}


-- End of files --


# Notes
- This is a simple console-based skeleton. For production, use password hashing (BCrypt), input validation, connection pooling, and an MVC web framework (Spring Boot + JPA) for a full portal.
- Update DBConnection credentials before running.
- To compile: place files under proper package folders, add MySQL JDBC driver to classpath, run Main.
# GUVI
Database Code (jobportal.sql)
CREATE DATABASE jobportal;
USE jobportal;


//CREATE TABLE users (
id INT AUTO_INCREMENT PRIMARY KEY,
name VARCHAR(100),
email VARCHAR(100) UNIQUE,
password VARCHAR(100),
role VARCHAR(20)
);


CREATE TABLE jobs (
id INT AUTO_INCREMENT PRIMARY KEY,
title VARCHAR(100),
company VARCHAR(100),
description TEXT
);
//DBConnection.java
package dao;
import java.sql.*;


public class DBConnection {
public static Connection getConnection() {
Connection con = null;
try {
Class.forName("com.mysql.cj.jdbc.Driver");
con = DriverManager.getConnection(
"jdbc:mysql://localhost:3306/jobportal",
"root",
"password"
);
} catch (Exception e) {
e.printStackTrace();
}
return con;
}
}
//User.java
package model;
public class User {
private String name, email, password, role;
public String getName() { return name; }
public void setName(String name) { this.name = name; }
public String getEmail() { return email; }
public void setEmail(String email) { this.email = email; }
public String getPassword() { return password; }
public void setPassword(String password) { this.password = password; }
public String getRole() { return role; }
public void setRole(String role) { this.role = role; }
}
//Job.java
package model;
public class Job {
private String title, company, description;
public String getTitle() { return title; }
public void setTitle(String title) { this.title = title; }
public String getCompany() { return company; }
public void setCompany(String company) { this.company = company; }
public String getDescription() { return description; }
public void setDescription(String description) { this.description = description; }
}
//UserDAO.java
package dao;
import java.sql.*;
import model.User;


public class UserDAO {
public static boolean register(User u) {
try {
Connection con = DBConnection.getConnection();
PreparedStatement ps = con.prepareStatement(
"INSERT INTO users(name,email,password,role) VALUES(?,?,?,?)"
);
ps.setString(1, u.getName());
ps.setString(2, u.getEmail());
ps.setString(3, u.getPassword());
ps.setString(4, u.getRole());
return ps.executeUpdate() > 0;
} catch (Exception e) {
return false;
}
}
public static boolean login(String email, String password) {
try {
Connection con = DBConnection.getConnection();
PreparedStatement ps = con.prepareStatement(
"SELECT * FROM users WHERE email=? AND password=?"
);
ps.setString(1, email);
ps.setString(2, password);
ResultSet rs = ps.executeQuery();
return rs.next();
} catch (Exception e) {
return false;
}
}
}
//JobDAO.java
package dao;
import java.sql.*;
import model.Job;


public class JobDAO {
public static boolean addJob(Job j) {
try {
Connection con = DBConnection.getConnection();
PreparedStatement ps = con.prepareStatement(
"INSERT INTO jobs(title,company,description) VALUES(?,?,?)"
);
ps.setString(1, j.getTitle());
ps.setString(2, j.getCompany());
ps.setString(3, j.getDescription());
return ps.executeUpdate() > 0;
} catch (Exception e) {
return false;
}
}
}
//LoginServlet.java
package controller;
import java.io.*;
import javax.servlet.*;
import javax.servlet.http.*;
import dao.UserDAO;


public class LoginServlet extends HttpServlet {
protected void doPost(HttpServletRequest req, HttpServletResponse res)
throws ServletException, IOException {
String email = req.getParameter("email");
String password = req.getParameter("password");


if (UserDAO.login(email, password)) {
res.sendRedirect("dashboard.jsp");
} else {
res.sendRedirect("error.jsp");
}
}
}
//RegisterServlet.java
package controller;
import java.io.*;
import javax.servlet.*;
import javax.servlet.http.*;
import model.User;
import dao.UserDAO;


public class RegisterServlet extends HttpServlet {
protected void doPost(HttpServletRequest req, HttpServletResponse res)
throws ServletException, IOException {
User u = new User();
u.setName(req.getParameter("name"));
u.setEmail(req.getParameter("email"));
u.setPassword(req.getParameter("password"));
u.setRole(req.getParameter("role"));


if (UserDAO.register(u)) {
res.sendRedirect("login.jsp");
} else {
res.sendRedirect("error.jsp");
}
}
}
//JobServlet.java
package controller;
import java.io.*;
import javax.servlet.*;
import javax.servlet.http.*;
import model.Job;
import dao.JobDAO;


public class JobServlet extends HttpServlet {
protected void doPost(HttpServletRequest req, HttpServletResponse res)
throws ServletException, IOException {
Job j = new Job();
j.setTitle(req.getParameter("title"));
j.setCompany(req.getParameter("company"));
j.setDescription(req.getParameter("description"));


if (JobDAO.addJob(j)) {
res.sendRedirect("dashboard.jsp");
} else {
res.sendRedirect("error.jsp");
}
}
}
