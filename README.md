# ECUT博客

## 1.操作指南

### 1.配置Tomcat

配置外部源`uploads`，将工件应用程序上下文部分修改为"/"。

### 2.配置数据库

​	创建数据库`ecut_blog`，运行`ecut_blog.sql`脚本自动建库。`src/main/resources/db.properties`配置本地数据库自己用户名，密码。

### 3.配置本地文件上传位置

​	在`com/wwey/ssm/blog/controller/admin/UploadFileController.java`下找到`public final String rootPath =`后配置本地`uploads`位置。

## 2.部分内容展示及源码解析

### 1.博客管理模块

![](https://github.com/1626901167/ECUTBolg/blob/main/img/1.png)

​	后台文章列表显示：该函数是一个后台文章列表显示的功能。它接收请求参数pageIndex（页码，默认为1）、pageSize（每页显示数量，默认为10）和status（文章状态），并根据这些参数查询文章列表。

​	函数首先创建一个HashMap对象criteria用于存储查询条件。如果status参数不为空，则将status放入criteria中，并构建相应的分页查询URL。然后从session中获取用户信息，判断用户角色，如果不是管理员，则在criteria中加入userId查询条件，以实现用户只能查询自己的文章的限制。

​	接下来，调用articleService的pageArticle方法进行分页查询，并将查询结果放入model中，最后返回视图名"Admin/Article/index"，即渲染的文章列表页面。

```java
@RequestMapping(value = "")
    public String index(@RequestParam(required = false, defaultValue = "1") Integer pageIndex,
                        @RequestParam(required = false, defaultValue = "10") Integer pageSize,
                        @RequestParam(required = false) String status, Model model,
                        HttpSession session) {
        HashMap<String, Object> criteria = new HashMap<>(1);
        if (status == null) {
            model.addAttribute("pageUrlPrefix", "/admin/article?pageIndex");
        } else {
            criteria.put("status", status);
            model.addAttribute("pageUrlPrefix", "/admin/article?status=" + status + "&pageIndex");
        }

        User user = (User) session.getAttribute("user");
        if (!UserRole.ADMIN.getValue().equals(user.getUserRole())) {
            // 用户查询自己的文章, 管理员查询所有的
            criteria.put("userId", user.getUserId());
        }
        PageInfo<Article> articlePageInfo = articleService.pageArticle(pageIndex, pageSize, criteria);
        model.addAttribute("pageInfo", articlePageInfo);
        return "Admin/Article/index";
    }

```

​	后台添加文章页面显示：该函数是一个Java方法，用于处理后台添加文章页面的显示。它通过调用categoryService.listCategory()和tagService.listTag()分别获取分类列表和标签列表，并将它们添加到Model中，然后返回一个字符串路径"Admin/Article/insert"，该路径指向文章添加页面的视图。

```java
@RequestMapping(value = "/insert")
    public String insertArticleView(Model model) {
        List<Category> categoryList = categoryService.listCategory();
        List<Tag> tagList = tagService.listTag();
        model.addAttribute("categoryList", categoryList);
        model.addAttribute("tagList", tagList);
        return "Admin/Article/insert";
    }

```

​	后台添加文章提交操作：该函数是一个后台添加文章的提交操作，使用POST请求访问/insertSubmit路径。函数首先从HttpSession中获取当前用户信息，然后根据用户提交的文章参数创建一个Article对象，并设置文章的标题、摘要、缩略图、内容、状态等属性。摘要内容通过截取文章内容前150个字符或整个内容生成。接着，根据文章参数填充分类和标签信息，并调用articleService的insertArticle方法将文章插入数据库。最后，函数重定向到/admin/article页面。

```java
@RequestMapping(value = "/insertSubmit", method = RequestMethod.POST)
    public String insertArticleSubmit(HttpSession session, ArticleParam articleParam) {
        Article article = new Article();
        //用户ID
        User user = (User) session.getAttribute("user");
        if (user != null) {
            article.setArticleUserId(user.getUserId());
        }
        article.setArticleTitle(articleParam.getArticleTitle());
        //文章摘要
        int summaryLength = 150;
        String summaryText = HtmlUtil.cleanHtmlTag(articleParam.getArticleContent());
        if (summaryText.length() > summaryLength) {
            String summary = summaryText.substring(0, summaryLength);
            article.setArticleSummary(summary);
        } else {
            article.setArticleSummary(summaryText);
        }
        article.setArticleThumbnail(articleParam.getArticleThumbnail());
        article.setArticleContent(articleParam.getArticleContent());
        article.setArticleStatus(articleParam.getArticleStatus());
        //填充分类
        List<Category> categoryList = new ArrayList<>();
        if (articleParam.getArticleChildCategoryId() != null) {
            categoryList.add(new Category(articleParam.getArticleParentCategoryId()));
        }
        if (articleParam.getArticleChildCategoryId() != null) {
            categoryList.add(new Category(articleParam.getArticleChildCategoryId()));
        }
        article.setCategoryList(categoryList);
        //填充标签
        List<Tag> tagList = new ArrayList<>();
        if (articleParam.getArticleTagIds() != null) {
            for (int i = 0; i < articleParam.getArticleTagIds().size(); i++) {
                Tag tag = new Tag(articleParam.getArticleTagIds().get(i));
                tagList.add(tag);
            }
        }
        article.setTagList(tagList);

        articleService.insertArticle(article);
        return "redirect:/admin/article";
    }

```

​	删除文章：该函数是一个Java方法，用于删除文章。它通过HTTP POST请求访问位于/delete/{id}的URL，其中{id}是文章的ID。方法使用@PathVariable注解从URL中获取ID，并通过HttpSession获取当前用户信息。然后，它从文章服务中获取具有给定ID和状态的文章对象。如果文章不存在，则方法直接返回。接下来，它验证当前用户是否具有删除文章的权限：如果是管理员或文章的作者，则可以执行删除操作，否则方法将直接返回。最后，如果用户具有权限，则调用文章服务的方法来删除文章。

```java
@RequestMapping(value = "/delete/{id}", method = RequestMethod.POST)
    public void deleteArticle(@PathVariable("id") Integer id, HttpSession session) {
        Article dbArticle = articleService.getArticleByStatusAndId(null, id);
        if (dbArticle == null) {
            return;
        }
        User user = (User) session.getAttribute("user");
        // 如果不是管理员，访问其他用户的数据，则跳转403
        if (!Objects.equals(dbArticle.getArticleUserId(), user.getUserId()) && !Objects.equals(user.getUserRole(), UserRole.ADMIN.getValue())) {
            return;
        }
        articleService.deleteArticle(id);
    }

```

![](https://github.com/1626901167/ECUTBolg/blob/main/img/3.png)

​	编辑文章页面显示：该函数是一个Java方法，用于处理编辑文章页面的显示请求。它接收一个文章ID作为路径参数，使用该ID从文章服务中获取文章对象。如果文章对象不存在，则重定向到404页面。接着，从会话中获取当前用户对象，并进行权限验证，如果当前用户不是文章的作者且不是管理员，则重定向到403权限拒绝页面。通过模型对象将文章对象传递给前端页面，并从分类服务和标签服务中获取分类列表和标签列表，也一并传递给前端页面。最后，返回编辑文章页面的视图路径

```java
@RequestMapping(value = "/edit/{id}")
    public String editArticleView(@PathVariable("id") Integer id, Model model, HttpSession session) {

        Article article = articleService.getArticleByStatusAndId(null, id);
        if (article == null) {
            return "redirect:/404";
        }
        User user = (User) session.getAttribute("user");
        // 如果不是管理员，访问其他用户的数据，则跳转403
        if (!Objects.equals(article.getArticleUserId(), user.getUserId()) && !Objects.equals(user.getUserRole(), UserRole.ADMIN.getValue())) {
            return "redirect:/403";
        }

        model.addAttribute("article", article);

        List<Category> categoryList = categoryService.listCategory();
        model.addAttribute("categoryList", categoryList);

        List<Tag> tagList = tagService.listTag();
        model.addAttribute("tagList", tagList);

        return "Admin/Article/edit";
    }

```

### 2.用户管理模块

![](https://github.com/1626901167/ECUTBolg/blob/main/img/2.png)

​	后台用户列表显示：该函数是一个Java后台用户列表显示函数，使用了Spring MVC框架的ModelAndView类。函数首先创建了一个ModelAndView对象，然后调用userService.listUser()方法获取用户列表并将其添加到ModelAndView对象中，最后设置视图名称为"Admin/User/index"并返回ModelAndView对象。该函数的作用是返回一个用户列表的模型和视图，用于在后台管理界面显示用户列表。

```java
@RequestMapping(value = "")
    public ModelAndView userList() {
        ModelAndView modelandview = new ModelAndView();

        List<User> userList = userService.listUser();
        modelandview.addObject("userList", userList);

        modelandview.setViewName("Admin/User/index");
        return modelandview;

}

```

​	后台添加用户页面显示：这个Java函数是一个后台添加用户页面的显示方法。它通过@RequestMapping注解将URL映射到/insert上，当用户访问这个URL时，会调用这个方法。方法内部创建了一个ModelAndView对象，并设置了视图名称为"Admin/User/insert"，然后返回这个ModelAndView对象。视图名称指定了要显示的页面的位置，所以这个方法的作用是将用户重定向到后台添加用户的页面。

```java
@RequestMapping(value = "/insert")
    public ModelAndView insertUserView() {
        ModelAndView modelAndView = new ModelAndView();
        modelAndView.setViewName("Admin/User/insert");
        return modelAndView;
    }

```

​	检查用户名是否存在：该函数是一个Java方法，用于检查给定的用户名是否已经存在。它通过HttpServletRequest对象接收前端发送的请求，并从请求中获取用户名和用户ID。然后，它调用userService.getUserByName(username)方法来查询数据库中是否存在该用户名对应的用户。如果存在，它会判断该用户ID是否与请求中提供的ID相同，如果不同，则将返回一个包含错误代码和消息的JSON字符串，提示用户名已存在。如果不存在，它将返回一个包含成功代码和空消息的JSON字符串。最后，该方法将结果字符串返回给前端。

```java
@RequestMapping(value = "/checkUserName", method = RequestMethod.POST, produces = {"text/plain;charset=UTF-8"})
    @ResponseBody
    public String checkUserName(HttpServletRequest request) {
        Map<String, Object> map = new HashMap<String, Object>();
        String username = request.getParameter("username");
        User user = userService.getUserByName(username);
        int id = Integer.valueOf(request.getParameter("id"));
        //用户名已存在,但不是当前用户(编辑用户的时候，不提示)
        if (user != null) {
            if (user.getUserId() != id) {
                map.put("code", 1);
                map.put("msg", "用户名已存在！");
            }
        } else {
            map.put("code", 0);
            map.put("msg", "");
        }
        String result = new JSONObject(map).toString();
        return result;
    }

```

​	删除用户：该函数是一个Java方法，用于删除指定ID的用户。它通过@RequestMapping注解将URL路径与方法绑定，使用@PathVariable注解将URL中的{id}参数绑定到方法参数id上。方法调用userService.deleteUser(id)来删除用户，并通过返回字符串"redirect:/admin/user"实现页面重定向到用户管理页面。

```java
@RequestMapping(value = "/delete/{id}")
    public String deleteUser(@PathVariable("id") Integer id) {
        userService.deleteUser(id);
        return "redirect:/admin/user";
    }

```

​	编辑用户页面显示：该函数是一个Java控制器方法，用于处理编辑用户页面的请求。函数通过接收一个用户ID作为参数，使用userService.getUserById(id)方法获取对应的用户信息，并将用户信息添加到ModelAndView对象中，然后将页面重定向到Admin/User/edit。最终返回ModelAndView对象。具体流程如下：创建一个ModelAndView对象。根据传入的ID使用userService.getUserById(id)方法获取User对象。将获取到的User对象添加到ModelAndView对象中，键为"user"。设置ModelAndView对象的视图名称为"Admin/User/edit"。返回ModelAndView对象。

```java
@RequestMapping(value = "/edit/{id}")
    public ModelAndView editUserView(@PathVariable("id") Integer id) {
        ModelAndView modelAndView = new ModelAndView();

        User user = userService.getUserById(id);
        modelAndView.addObject("user", user);

        modelAndView.setViewName("Admin/User/edit");
        return modelAndView;
    }

```

​	编辑用户提交：该函数是一个Java方法，用于处理用户提交的编辑信息，并将更新后的用户信息保存到数据库中。该方法使用了Spring MVC框架的@RequestMapping注解，指定了请求的URL和请求方法。方法接收一个User类型的参数，该参数为要更新的用户信息。方法调用userService.updateUser(user)来更新用户信息，并返回一个重定向的URL，将页面重定向到管理员用户列表页面。

```java
@RequestMapping(value = "/editSubmit", method = RequestMethod.POST)
    public String editUserSubmit(User user) {
        userService.updateUser(user);
        return "redirect:/admin/user";
    }

```

### 3.评论管理模块

![4](https://github.com/1626901167/ECUTBolg/blob/main/img/4.png)

​	发送的评论：该函数是一个Java方法，用于处理评论列表页面的请求。它接收请求参数pageIndex和pageSize来指定需要查询的页码和页大小。该方法从HttpSession中获取当前用户对象，并根据用户角色来构建查询条件。如果用户不是管理员，则查询该用户自己的评论；否则，查询所有评论。然后，调用commentService的listCommentByPage方法来获取评论分页信息，并将结果添加到Model中，以便在页面上显示。最后，返回一个视图名称"Admin/Comment/index"，表示需要渲染的页面。

```java
@RequestMapping(value = "")
    public String commentList(@RequestParam(required = false, defaultValue = "1") Integer pageIndex,
                              @RequestParam(required = false, defaultValue = "10") Integer pageSize,
                              HttpSession session,
                              Model model) {
        User user = (User) session.getAttribute("user");
        HashMap<String, Object> criteria = new HashMap<>();
        if (!UserRole.ADMIN.getValue().equals(user.getUserRole())) {
            // 用户查询自己的文章, 管理员查询所有的
            criteria.put("userId", user.getUserId());
        }
        PageInfo<Comment> commentPageInfo = commentService.listCommentByPage(pageIndex, pageSize, criteria);
        model.addAttribute("pageInfo", commentPageInfo);
        model.addAttribute("pageUrlPrefix", "/admin/comment?pageIndex");
        return "Admin/Comment/index";
    }

```

​	收到的评论：该函数是一个Java控制器方法，用于处理评论页面中“我收到的评论”这一功能的请求。其功能如下：接收请求参数：页码(pageIndex)和页大小(pageSize)，并提供默认值。通过HttpSession获取当前用户对象(User)并打印其信息。调用commentService的listReceiveCommentByPage方法，根据用户ID、页码和页大小查询用户收到的评论信息。将查询结果(PageInfo<Comment>)添加到模型(Model)中，以便在页面上显示。添加分页链接的URL前缀(/admin/comment?pageIndex)到模型中。返回视图名称("Admin/Comment/index")，即评论页面的路径。

```java
@RequestMapping(value = "/receive")
    public String myReceiveComment(@RequestParam(required = false, defaultValue = "1") Integer pageIndex,
                                   @RequestParam(required = false, defaultValue = "10") Integer pageSize,
                                   HttpSession session,
                                   Model model) {
        User user = (User) session.getAttribute("user");
        System.out.println(user);
        PageInfo<Comment> commentPageInfo = commentService.listReceiveCommentByPage(pageIndex, pageSize, user.getUserId());
        model.addAttribute("pageInfo", commentPageInfo);
        model.addAttribute("pageUrlPrefix", "/admin/comment?pageIndex");
        return "Admin/Comment/index";
    }

```

​	添加评论：该函数是一个Java方法，用于添加评论。它使用了Spring MVC框架的@RequestMapping注解，指定了请求的URL路径为/insert，请求方法为POST，返回类型为text/plain;charset=UTF-8。方法参数包括HttpServletRequest request，Comment comment和HttpSession session。其中，request用于获取客户端请求信息，comment是评论对象，session用于获取会话信息。

```java
 @ResponseBody
    public void insertComment(HttpServletRequest request, Comment comment, HttpSession session) {
        User user = (User) session.getAttribute("user");
        Article article = articleService.getArticleByStatusAndId(null, comment.getCommentArticleId());
        if (article == null) {
            return;
        }

        //添加评论
        comment.setCommentUserId(user.getUserId());
        comment.setCommentIp(MyUtils.getIpAddr(request));
        comment.setCommentCreateTime(new Date());
        commentService.insertComment(comment);
        //更新文章的评论数
        articleService.updateCommentCount(article.getArticleId());
    }

```

​	删除评论：该函数是一个Java方法，用于删除评论。它通过请求映射注解@RequestMapping(value = "/delete/{id}")将请求路径与方法绑定，接收一个评论ID，并从HttpSession中获取当前用户信息。方法首先根据ID获取评论对象，然后判断当前用户是否为管理员或评论的发布者，如果不是则无权限删除并返回。接着，方法会删除指定ID的评论，并递归删除其所有子评论。最后，方法更新所评论的文章的评论数量。

```java
@RequestMapping(value = "/delete/{id}")
    public void deleteComment(@PathVariable("id") Integer id, HttpSession session) {
        Comment comment = commentService.getCommentById(id);
        User user = (User) session.getAttribute("user");
        // 如果不是管理员，访问其他用户的数据，没有权限
        if (!Objects.equals(user.getUserRole(), UserRole.ADMIN.getValue()) && !Objects.equals(comment.getCommentUserId(), user.getUserId())) {
            return;
        }
        //删除评论
        commentService.deleteComment(id);
        //删除其子评论
        List<Comment> childCommentList = commentService.listChildComment(id);
        for (int i = 0; i < childCommentList.size(); i++) {
            commentService.deleteComment(childCommentList.get(i).getCommentId());
        }
        //更新文章的评论数
        Article article = articleService.getArticleByStatusAndId(null, comment.getCommentArticleId());
        articleService.updateCommentCount(article.getArticleId());
    }

```

​		编辑评论页面显示：该函数是一个Java方法，用于处理编辑评论页面的显示。它接收一个整数类型的id参数，并通过模型和会话对象来传递数据和控制流程。通过@RequestMapping注解，将该方法与URL路径/edit/{id}进行映射。使用@PathVariable注解将URL路径中的{id}绑定到方法参数id上，获取指定id的评论。从会话对象中获取当前用户信息，并判断用户角色是否为管理员。如果用户不是管理员，则重定向到403权限拒绝页面。如果用户是管理员，则从评论服务中获取指定id的评论，并将其添加到模型对象中。返回对应的视图路径Admin/Comment/edit，用于渲染编辑评论的页面。

```java
@RequestMapping(value = "/edit/{id}")
    public String editCommentView(@PathVariable("id") Integer id, Model model, HttpSession session) {
        // 没有权限操作,只有管理员可以操作
        User user = (User) session.getAttribute("user");
        if (!Objects.equals(user.getUserRole(), UserRole.ADMIN.getValue())) {
            return "redirect:/403";
        }
        Comment comment = commentService.getCommentById(id);
        model.addAttribute("comment", comment);
        return "Admin/Comment/edit";
    }

```





###### 特别鸣谢，参考model(https://github.com/saysky/ForestBlog)

