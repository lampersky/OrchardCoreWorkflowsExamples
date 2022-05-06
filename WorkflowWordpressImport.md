## OrchardCore Workflow WordPress Importer

Let's create a simple Orchard Core Workflows app which allow us to import some data from WordPress blog using WordPress Rest Api

If you are not familiar with WP Rest Api, you can read more here: https://developer.wordpress.org/rest-api/

This app is not a final "ready to use" product, it is just a proof of concept, and has many limitations.

It allows you to import some basic WordPress types like:
- **categories** (we will store them as terms),
- **tags** (we will store them as terms),
- **pages** (we will store them as ContentItems),
- **posts** (we will store them as ContentItems).

Requirements:
- because imports usually take a while to finish, it would be great if we could run it in a fire-and-forget manner,
- only one import can run at the same time,
- it would be good to be able to monitor import progress,
- present some basic import summary when import is done,
- same import can be run multiple times without creating duplicates,
- user can choose which WordPress types to import,
- build it in an extensible way, so we can add other WordPress types in the future.

### Steps

1. Before you will start, please make sure you are using **Blog** recipe.

2. Create a few ContentTypes and one ContentPart:

   - **ImportState** - this ContentType holds basic informations about our import, like base url, what we want to import, where we want to save newly imported posts, categories or tags, also it contains field which tells us if import is finished or not, other than that this type includes ListPart which can accept items like **ImportItem** and **ImportTaxonomy**,
   - **ImportItem** - this ContentType holds informations about **ContentItem** import, it also includes ListPart which can accept **ImportProgressItem**,
   - **ImportTaxonomy** - this ContentType holds informations about **Term** import, it also includes ListPart which can accept **ImportProgressItem**,
   - **ImportProgressItem** - this ContentType holds progress of Terms/ContentItems import,
   - **ImportStart** - it will be used as our starting point for imports, it only contains AutoroutePart,
   - **ImportPart** - contains some common filds like Name, Progress, BaseUrl, InputType, Fields, OutputType, Page, PerPage.

3. Create new ContentItem of type **ImportStart** and set its Permalink as "import-blog",

4. Create new template, name it as follows "Content__ImportStart", add some simple form, we will complete missing form action later:

    ```liquid
    {% assign isAdmin = User | is_in_role:"Administrator" %}
    {% zone "Header" %}
        <header class="masthead" style="background-image: url('{{ "~/TheBlogTheme/assets/img/home-bg.jpg" | href }}')">
        <div class="container position-relative px-4 px-lg-5">
            <div class="row gx-4 gx-lg-5 justify-content-center">
                <div class="col-md-10 col-lg-8 col-xl-7">
                    <div class="site-heading">
                        {% if isAdmin %}
                            <h1>{{ Model.ContentItem.DisplayText }}</h1>
                            <form action="TODO" method="post">
                                <div class="form-row">
                                    <div class="form-group">
                                        <label for="wordpressBlogUrl">Blog url</label>
                                        <input type="url" class="form-control" id="wordpressBlogUrl" name="url" placeholder="https://your-blog-url-here.com/wp-json/wp/v2/">
                                    </div>
                                </div>
                                <div class="form-row">
                                    <div class="form-check form-check-inline">
                                        <input class="form-check-input" type="checkbox" id="tags" name="what" value="tags">
                                        <label class="form-check-label" for="tags">Tags</label>
                                    </div>
                                    <div class="form-check form-check-inline">
                                        <input class="form-check-input" type="checkbox" id="categories" name="what" value="categories">
                                        <label class="form-check-label" for="categories">Categories</label>
                                    </div>
                                    <div class="form-check form-check-inline">
                                        <input class="form-check-input" type="checkbox" id="pages" name="what" value="pages">
                                        <label class="form-check-label" for="pages">Pages</label>
                                    </div>
                                    <div class="form-check form-check-inline">
                                        <input class="form-check-input" type="checkbox" id="posts" name="what" value="posts">
                                        <label class="form-check-label" for="posts">Posts</label>
                                    </div>
                                </div>
                                <div class="form-row">
                                    <div class="col">
                                        <button type="submit" class="btn btn-primary mb-2">Begin import</button>
                                    </div>
                                </div>
                                {% antiforgerytoken %}
                            </form>
                        {% else %}
                            <div>Only user with "Administrator" role can perform import!</div>
                        {% endif %}
                    </div>
                </div>
            </div>
        </div>
    </header>
    {% endzone %}
    ```

It should look like this:

  ![image](https://user-images.githubusercontent.com/6403130/167016184-11f14fdb-f5b2-47e1-aa3d-49e42f834bfe.png)

5. We will need some SQL queries so, let's create them now:

    - **getActiveImportState** - returns active import details and progress if available, 

      (we could avoid those left joins by splitting this query into two separete queries)

      (**ContentItemIndex** is not used intentionally),

      ```sql
      SELECT api.ContentItemId as ListContentItemId, api.Path, tfi.ContentItemId, tfi.Text as Title, bfi.Boolean as Finished, max(p.Numeric) as Progress, ti.Numeric as TotalItems FROM
      AutoroutePartIndex as api LEFT JOIN
      ContainedPartIndex cpi on api.ContentItemId = cpi.ListContentItemId LEFT JOIN
      TextFieldIndex as tfi on tfi.DocumentId = cpi.DocumentId and tfi.ContentField = 'Name' LEFT JOIN
      BooleanFieldIndex as bfi on cpi.ListContentItemId = bfi.ContentItemId and bfi.ContentField = 'Finished' LEFT JOIN
      ContainedPartIndex as pr on pr.ListContentItemId = tfi.ContentItemId LEFT JOIN
      NumericFieldIndex as p on p.DocumentId = pr.DocumentId and p.ContentField = 'Progress' LEFT JOIN
      NumericFieldIndex as ti on ti.DocumentId = pr.DocumentId and ti.ContentField = 'TotalItems'
      WHERE api.Path = 'progress/latest' and api.Latest = 1 and api.Published = 1 
      group by api.ContentItemId, api.Path, tfi.ContentItemId, tfi.Text, bfi.Boolean, ti.Numeric;
      ```

    - **getContentItemById** - returns ContentItem (Document) for provided ContentItemId,

      ```sql
      SELECT cii.DocumentId FROM ContentItemIndex as cii
      where cii.ContentItemId = @ContentItemId:'' and Latest = 1 and Published = 1
      ```
    
    - **getTaxonomiesAndBlogIds** - returns ContentItemIds for Categories, Tags and Blog.

      ```sql
      select ContentItemId, DisplayText as Name
      from ContentItemIndex where Published = 1 and Latest = 1 and (ContentType = 'Blog' or ContentType = 'Taxonomy');
      ```
 
6. Now it is time to create first Workflow, which will handle incoming requests from our form, let's name it **PostStartImport**, please note this Workflow needs to be singleton:

    - it checks antiforgery token,
    - it checks if user has Administrator role,
    - it reads form data and performs some basic checking,
    - it queries db **getActiveImportState** for active import,
    - finally it creates new ContentItem of type **ImportState** and set it as homepage,
    - at this step you can copy URL from **Http Request Event** and paste it into **Content__ImportStart** template.

    ![image](https://user-images.githubusercontent.com/6403130/167021334-f2b2f989-ed8e-4d71-81a3-f1334b93d447.png)

7. If everything is ok, you should be able to execute workflow by visiting `/import-blog` and filling and submitting form.

8. Now, it is time to create another template, this time for **ImportState** ContentType,
   so the name should be "Content__ImportState", this template includes some basic javascript, which performs some GET request,
   we will provide URL of that request later

    ```liquid
    {% zone "Header" %}
        <header class="masthead" style="background-image: url('{{ "~/TheBlogTheme/assets/img/home-bg.jpg" | href }}')">
        <div class="container position-relative px-4 px-lg-5">
            <div class="row gx-4 gx-lg-5 justify-content-center">
                <div class="col-md-10 col-lg-8 col-xl-7">
                    <div class="site-heading">
                        <h1>{{ Model.ContentItem.DisplayText }}</h1>

                            <div class="card" style="background-color: #ffffff60; border-radius: 10px;">
                                <div class="card-body">

                                    <div id="preloader">
                                        <div class="d-flex flex-column align-items-center justify-content-center">
                                            <div class="spinner-border" role="status">
                                                <span class="sr-only">Loading...</span>
                                            </div>
                                            <div>
                                                Import will start shortly, it usually takes up to 60 seconds
                                            </div>
                                        </div>
                                    </div>

                                    <div id="progress" class="d-none">
                                        {% assign items = Model.ContentItem | list_items %}
                                        {% for item in items %}
                                            <div>
                                                {{ item | shape_build_display: "Summary" | shape_render}}
                                            </div>
                                        {% endfor %}
                                    </div>

                                    <ul id="finished" class="d-none list-group">
                                        <li class="list-group-item active">Import result</li>
                                    </ul>

                                </div>
                            </div>

                    </div>
                </div>
            </div>
        </div>
    </header>
    {% endzone %}

    {% unless Model.ContentItem.Content.Finished %}
        {% scriptblock at: "Foot" %}
            const addProgressBar = function(parent, item) {
                const container = document.createElement("div");
                container.innerHTML = `<div><div>${item.Title}</div><div class="progress"><div data-content-item-id="${item.ContentItemId}" class="progress-bar progress-bar-striped progress-bar-animated" role="progressbar" aria-valuenow="${item.Progress}" aria-valuemin="0" aria-valuemax="100" style="width: ${item.Progress}%;"></div></div></div>`;
                parent.appendChild(container);
            };

            const addResult = function(parent, item) {
                const container = document.createElement("li");
                container.classList.add("list-group-item");
                container.innerHTML = `Imported ${item.Title} : <b>${item.TotalItems}</b>`;
                parent.appendChild(container);
            };

            const refresh = function() {
                fetch("TODO")
                    .then(response => response.json())
                    .then(data => {
                        var overallProgress = 0;
                        const progressContainer = document.getElementById("progress");
                        data.items.forEach(function (item, index) {
                            const progress = item.Progress;
                            overallProgress += progress;
                            const el = document.querySelector(`div[data-content-item-id='${item.ContentItemId}']`);
                            if (el) {
                                el.style.width = `${progress}%`;
                                el.ariaValueNow = `${progress}`;
                            } else if (item.ContentItemId) {
                                addProgressBar(progressContainer, item);
                            }
                        });
                        overallProgress = data.items.length == 0 ? 0 : overallProgress / data.items.length;
                        if (overallProgress > 0) {
                            document.getElementById("preloader").classList.add("d-none");
                            document.getElementById("progress").classList.remove("d-none");
                        }
                        if (data.status === "starting") {
                            setTimeout(refresh, 10*1000);
                        } else if (data.status === "inProgress") {
                            setTimeout(refresh, 1000);
                        } else if (data.status === "finished") {
                            document.getElementById("preloader").classList.add("d-none");
                            document.getElementById("progress").classList.add("d-none");
                            document.getElementById("finished").classList.remove("d-none");
                            const resultContainer = document.getElementById("finished");
                            data.items.forEach(function (item, index) {
                                addResult(resultContainer, item);
                            });
                        }
                    });
            };

            window.addEventListener("DOMContentLoaded", function() {
                refresh();
            }, false);
        {% endscriptblock %}
    {% endunless %}

    {% styleblock at: "Foot" %}
        #finished > li:first-child {
            border-radius: 10px 10px 0px 0px;
        }
        #finished > li:last-child {
            border-radius: 0px 0px 10px 10px;
        }
    {% endstyleblock %}
    ```

This page will handle three different states of our import:
- starting,

  ![image](https://user-images.githubusercontent.com/6403130/167026069-4e5cb27b-c23d-4cbb-9355-964f0202ce68.png)

- in progress,

  ![image](https://user-images.githubusercontent.com/6403130/167025813-33c3dcab-50ad-4ece-ba58-b979e06ca947.png)

- finished.
  
  ![image](https://user-images.githubusercontent.com/6403130/167026389-52b942a3-82db-4b6e-a1a7-88acd46dcbf9.png)

9. We need two more templates **Content_Summary__ImportTaxonomy** and **Content_Summary__ImportStateItem**
  (we can't just use contentItemId as **id** attribute for div, because id value is an opaque string,
  but we can create our own attribute and store everything we want)

    ```liquid
    {% assign valueNow = Model.ImportPart.Progress.Value %}
    {% assign contentItemId = Model.ContentItem.ContentItemId %}
    <div>{{ Model.ContentItem | display_text }}</div>
    <div class="progress">
        <div data-content-item-id="{{ contentItemId }}" class="progress-bar progress-bar-striped progress-bar-animated" 
            role="progressbar" aria-valuenow="{{ valueNow }}" aria-valuemin="0" 
            aria-valuemax="100" style="width: {{ valueNow }}%"></div>
    </div>
    ```

    I've also modified a bit **Content_Summary__BlogPost** to properly display some HTML Entities included inside imported titles.

10. Now, it is time to create Workflow which will give us current import progress

    - it queries db **getActiveImportState** for active import,
    - at this step you can copy URL from Http Request Event and paste it into **Content__ImportState** template.

    ![image](https://user-images.githubusercontent.com/6403130/167027605-57cd9c8e-5d2d-40f3-8926-e36a8f789331.png)

11. Orchard Core **Blog** recipe is using **MarkdownBodyPart** for **BlogPost** ContentItem, we need to override default implementation,
    because we will be using **HtmlBodyPart** instead. So, create new template **Content__BlogPost** and use this content:

    ```liquid
    {% zone "Header" %}
        <!-- Page Header -->
        <!-- Set your background image for this header on the line below. -->
        {% assign imagePath = Model.ContentItem.Content.BlogPost.Image.Paths.first %}
        {% if imagePath == nil %}
            <header class="masthead" style="background-image: url('{{ "~/TheBlogTheme/assets/img/post-bg.jpg" | href }}')">
        {% else %}
            {% assign anchor = Model.ContentItem.Content.BlogPost.Image.Anchors.first %}
            <header class="masthead" style="background-image: url('{{ imagePath | asset_url | resize_url: profile:"banner", anchor: anchor }}')">
        {% endif %}
            <div class="container position-relative px-4 px-lg-5">
                <div class="row gx-4 gx-lg-5 justify-content-center">
                    <div class="col-md-10 col-lg-8 col-xl-7">
                        <div class="post-heading">
                            <h1>{{ Model.ContentItem.DisplayText | liquid }}</h1>
                            <h2 class="subheading">{{ Model.ContentItem.Content.BlogPost.Subtitle.Text }}</h2>
                            <span class="meta">
                                {% assign format = "MMMM dd, yyyy" | t %}
                                {% assign dateTime = "DateTime" | shape_new: utc: Model.ContentItem.CreatedUtc, format: format | shape_stringify %}
                                {{ "Posted by" | t }} <a href="#">{{ Model.ContentItem.Author }}</a> {{ "on {0}" | t: dateTime | raw }}
                            </span>
                        </div>
                    </div>
                </div>
            </div>
        </header>
    {% endzone %}

    {{ Model.Content.ContentsMetadata | shape_render }}
    {{ Model.Content.HtmlBodyPart | shape_render }}
    {{ Model.Content.BlogPost-Category | shape_render }}
    {{ Model.Content.BlogPost-Tags | shape_render }}

    {% styleblock at: "Foot" %}
        figure > img {
            height: 100%;
            width: 100%;
        }
    {% endstyleblock %}
    ```

    Some additional css is added, to tweak a bit images.

12. Now, we can create another workflow **OnImportStateCreated** which will execute everytime new **ImportState** ContentItem is created:
    
    - whole import process is a time taking task and we don't want to execute this workflow within HTTP request,
      that's why we are halting workflow at very beginning and waiting for TimerTrigger, once TimerTrigger event is fired,
      Workflow will continue, but this time "in the background",
    - this workflow creates few more ContentItems conditionally,
    - once everything is done, **ImportState** is marked as finished and **Blog** ContentItem is marked as Homepage,
    
    ![image](https://user-images.githubusercontent.com/6403130/167034573-9280fbf9-9d9a-4bb8-8410-adfea77683b7.png)

13. Create **OnImportItemCreated** Workflow which will execute everytime new **ImportItem** ContentItem is created:

    - it fetches items from Wordpress Rest api in a loop,
    - after page is fetched, new **ImportProgressItem** content item is created with current progress,
    - it checks ContentItem uniqueness, if ContentItem with same ID already exists it will be overwritten.

    ![image](https://user-images.githubusercontent.com/6403130/167035752-d3e0e4d6-1538-4752-bda2-a0bdc7d4bbfb.png)

14. Create **OnImportTaxonomyCreated** Workflow which will execute everytime new **ImportTaxonomy** ContentItem is created:

    - this Workflow is quite similar to the previous one, the only difference is that, we are not creating new terms in a loop,
      we are collecting them and saving once all pages are processed,
    - it checks term uniqueness,
    - because of **Scripting Module** limitations there are some hacks involved (lots of spread operators).

    ![image](https://user-images.githubusercontent.com/6403130/167035901-71d5625c-3fdf-4bad-b29d-573814b58c7d.png)

15. It is time for test, you can see the app in the action here:

    https://youtu.be/dLPNuBQ7yKQ
