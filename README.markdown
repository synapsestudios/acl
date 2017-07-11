Why Synapse ACL?
---------------

This project is a fork of the wonderful https://github.com/djvirgen/virgen-acl as a result of
this issue https://github.com/djvirgen/virgen-acl/issues/14. You should probably use virgen-acl
instead of this library unless you read through the issue and have the same desire to use
it this way.

The only real reason this is even a public repo is because we'll be using the public
npm registry to host this module and so the repo might as well be public as well.

Installation
------------

    npm install @synapsestudios/acl

Usage
-----

    // Load library
    var Acl = require("@synapsestudios/acl").Acl
      , acl = new Acl();

    // Set up roles
    acl.addRole("guest");                     // guest user, inherits from no one
    acl.addRole("member", "guest");           // member inherits permissions from guest
    acl.addRole("admin");                     // Admin inherits from no one

    // Set up resources
    acl.addResource("blog");                  // blog resource, inherits no resources

    // Set up access rules (LIFO)
    acl.deny();                               // deny all by default
    acl.allow("admin");                       // allow admin access to everything
    acl.allow("member", "blog", "comment");   // allow members to comment on blogs
    acl.allow(null, "blog", "view");          // allow everyone to view the blogs
    acl.allow("guest", "blog", ["list", "search"]) // supports arrays of actions

    // Query the ACL
    acl.query("member", "blog", "comment", function(err, allowed) {
      if (allowed) {
        // commenting allowed!
      } else {
        // no commenting allowed!
      }
    });

    // supports multiple roles in query
    acl.query(["member", "admin"], "blog", "create", function(err, allowed) {
        if (allowed) {
          // creating allowed!
        } else {
          // no creating allowed!
        }
    });

Role and Resource Discovery
---------------------------

If you are more of an object-oriented programmer and prefer to use objects
to represent your roles and resources, then you're in luck! Virgen-ACL can
discover roles and resources from your objects so long as your role objects
contain the property `role_id` OR a function `getRoleId()` and your resource
objects contain the property `resource_id` OR a function `getResourceId()`.
Valid value types for role_ids are string, an array of strings, or `null`. Valid
value types for resource_ids are `null` or strings.
Here's an example of how that might work:

    // User class
    var User = (function(){
      User = function(attribs) {
        this.id = attribs.id || null;
      }

      // getRoleId gets access to the resource so it can return
      // different roles depending on the resource if you choose
      User.prototype.getRoleId = function(resource) {
        if (this.id) {
          return "member"; // members have an ID
        } else {
          return "guest"; // all other users are guests
        }
      }

      return User;
    })();

    // Blog class
    var Blog = (function(){
      Blog = function(attribs) {
        this.resource_id = "blog";
        this.status = attribs.status || "draft";
      };

      return Blog;
    })();

    var userA = new User();
    userA.getRoleId(); // returns "guest"
    var userB = new User({id: 123});
    userB.getRoleId(); // return "member"

    var blog = new Blog();
    blog.resource_id; // set to "blog"

    // Set up ACL
    var acl = new Acl();
    acl.addRole("guest");                   // guest inherits from no one
    acl.addRole("member", "guest");         // member inherits from guest
    acl.allow("guest", "blog", "view");     // guests allowed to view blog
    acl.allow("member", "blog", "comment"); // member allowed to comment on blog

    acl.query(userA, blog, "view", function(err, allowed) {
      // userA is a guest and can view blogs
      assert(allowed == true);
    });

    acl.query(userA, blog, "comment", function(err, allowed) {
      // userA is a guest and cannot comment on blogs
      assert(allowed == false);
    });

    acl.query(userB, blog, "view", function(err, allowed) {
      // userB is a member and inherits view permission from guest
      assert(allowed == true);
    });

    acl.query(userB, blog, "comment", function(err, allowed) {
      // userB is a member and has permission to comment on blogs
      assert(allowed == false);
    });

Custom Assertions
-----------------

Sometimes you need more complex rules when determining access. Custom
assertions can be provided to perform additional logic on each matching
ACL query:

    acl.allow("member", "blog", "edit", function(err, role, resource, action, result, next) {
      // Use next() if unable to determine permission based on provided arguments
      if (!(role instanceof User) || !(resource instanceof Blog))
        return next();

      if (role.id == resource.user_id) {
        // resource belongs to this role, allow editing
        result(null, true);
      } else {
        // resource does not belong to this role, do not allow editing
        result(null, false);
      }
    });

    var userA = new User({id: 123});
    assert(userA.id == 123);
    var userB = new User({id: 456});
    assert(userB.id == 456);
    var blog = new Blog({user_id: 123});
    assert(blog.user_id == 123);

    // userA can edit this blog because the blog's user ID matches the userA's ID
    acl.query(userA, blog, 'edit', function(err, allowed) {
      assert(allowed == true);
    });

    // However userB cannot edit this blog
    acl.query(userB, blog, 'edit', function(err, allowed) {
      assert(allowed == false);
    });
