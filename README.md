# echoperm [![Build Status](https://travis-ci.org/xyproto/echoperm.svg?branch=master)](https://travis-ci.org/xyproto/echoperm) [![GoDoc](https://godoc.org/github.com/xyproto/echoperm?status.svg)](http://godoc.org/github.com/xyproto/echoperm)


Middleware for [echo](https://github.com/labstack/echo) for handling users and permissions. Requires a Redis server and Go >= 1.7.

* Look into [permissions2](https://github.com/xyproto/permissions2) for more configuration options and documentation for the userstate struct.
* Look into [permissionbolt](https://github.com/xyproto/permissionbolt) for an alternative that stores the information in a database file instead.

### Usage

*Pseudocode*:

~~~go
e := echo.New()

redisHost := "localhost"
redisPassword := "hunter1"
permissionDenied := "Permission denied!"
epm, userstate, err := echoperm.Middleware(redisHost, redisPassword, permissionDenied)
if err != nil {
    ...
}
e.Use(epm)

e.Get("/", echo.HandlerFunc(func(c echo.Context) error {
    // Do things with userstate
    userstate.DoThings()
    ...
}))
~~~

### Example

~~~go
package main

import (
	"bytes"
	"fmt"
	"log"
	"net/http"
	"strings"

	"github.com/labstack/echo"
	"github.com/labstack/echo/middleware"
	"github.com/xyproto/echoperm"
)

func main() {
	e := echo.New()

	// Blank slate, no default permissions
	//perm.Clear()

	// Logging middleware
	e.Use(middleware.Logger())

	// Enable the permissions middleware, must come before recovery
	epm, userstate, err := echoperm.Middleware("localhost", "", "Permission denied!")
	if err != nil {
		log.Fatal(err)
	}
	e.Use(epm)

	// Recovery middleware
	e.Use(middleware.Recover())

	e.GET("/", echo.HandlerFunc(func(c echo.Context) error {
		var buf bytes.Buffer
		b2s := map[bool]string{false: "false", true: "true"}
		buf.WriteString("Has user bob: " + b2s[userstate.HasUser("bob")] + "\n")
		buf.WriteString("Logged in on server: " + b2s[userstate.IsLoggedIn("bob")] + "\n")
		buf.WriteString("Is confirmed: " + b2s[userstate.IsConfirmed("bob")] + "\n")
		buf.WriteString("Username stored in cookies (or blank): " + userstate.Username(c.Request()) + "\n")
		buf.WriteString("Current user is logged in, has a valid cookie and *user rights*: " + b2s[userstate.UserRights(c.Request())] + "\n")
		buf.WriteString("Current user is logged in, has a valid cookie and *admin rights*: " + b2s[userstate.AdminRights(c.Request())] + "\n")
		buf.WriteString("\nTry: /register, /confirm, /remove, /login, /logout, /makeadmin, /clear, /data and /admin")
		return c.String(http.StatusOK, buf.String())
	}))

	e.GET("/register", echo.HandlerFunc(func(c echo.Context) error {
		userstate.AddUser("bob", "hunter1", "bob@zombo.com")
		return c.String(http.StatusOK, fmt.Sprintf("User bob was created: %v\n", userstate.HasUser("bob")))
	}))

	e.GET("/confirm", echo.HandlerFunc(func(c echo.Context) error {
		userstate.MarkConfirmed("bob")
		return c.String(http.StatusOK, fmt.Sprintf("User bob was confirmed: %v\n", userstate.IsConfirmed("bob")))
	}))

	e.GET("/remove", echo.HandlerFunc(func(c echo.Context) error {
		userstate.RemoveUser("bob")
		return c.String(http.StatusOK, fmt.Sprintf("User bob was removed: %v\n", !userstate.HasUser("bob")))
	}))

	e.GET("/login", echo.HandlerFunc(func(c echo.Context) error {
		// Headers will be written, for storing a cookie
		userstate.Login(c.Response(), "bob")
		return c.String(http.StatusOK, fmt.Sprintf("bob is now logged in: %v\n", userstate.IsLoggedIn("bob")))
	}))

	e.GET("/logout", echo.HandlerFunc(func(c echo.Context) error {
		userstate.Logout("bob")
		return c.String(http.StatusOK, fmt.Sprintf("bob is now logged out: %v\n", !userstate.IsLoggedIn("bob")))
	}))

	e.GET("/makeadmin", echo.HandlerFunc(func(c echo.Context) error {
		userstate.SetAdminStatus("bob")
		return c.String(http.StatusOK, fmt.Sprintf("bob is now administrator: %v\n", userstate.IsAdmin("bob")))
	}))

	e.GET("/clear", echo.HandlerFunc(func(c echo.Context) error {
		userstate.ClearCookie(c.Response())
		return c.String(http.StatusOK, "Clearing cookie")
	}))

	e.GET("/data", echo.HandlerFunc(func(c echo.Context) error {
		return c.String(http.StatusOK, "user page that only logged in users must see!")
	}))

	e.GET("/admin", echo.HandlerFunc(func(c echo.Context) error {
		var buf bytes.Buffer
		buf.WriteString("super secret information that only logged in administrators must see!\n\n")
		if usernames, err := userstate.AllUsernames(); err == nil {
			buf.WriteString("list of all users: " + strings.Join(usernames, ", "))
		}
		return c.String(http.StatusOK, buf.String())
	}))

	// Serve
	e.Start(":3000")
}
~~~

Online API Documentation
------------------------

* [echoperm](http://godoc.org/github.com/xyproto/echoperm)
* [permissions2](http://godoc.org/github.com/xyproto/permissions2)


General information
-------------------

* Version: 2.0
* License: MIT
* Alexander F Rødseth &lt;xyproto@archlinux.org&gt;

