# Sitecore ContentLanguage Switcher

ContentLanguage.Switcher is a small module to switch automatically content language in Content Editor base on a site setting.


Sitecore allows us to set site language (or contentLanguage) in the site configuration (see [documentation](https://doc.sitecore.com/SdnArchive/Articles/Administration/Configuring%20Multiple%20Sites/Adding%20New%20Site/site%20Attribute%20Properties.html)). What if we had possibility, to automatically change content language in Content Manager depending on the selected site? In fact, with some simple customization we can easily achieve that!

Let me describe step by step how to do it.

First, we need to figure out how context change is handled in Content Tree. The best starting point will be to check Content Manager page located in `sitecore\shell\Applications\Content Manager\Default.aspx`.
![Image](https://user-images.githubusercontent.com/1537372/71258682-17f00200-2337-11ea-8e88-be25741a9ddf.png)
Here, we can see which class is used as CodeBeside (`ContentEditorForm`) and that there is a placeholder defined for Content Tree, so we are on good track - it's time to do some digging in code. You will probably notice that many things are happening here, but lucky for us we need to put interest only on two methods:

First, which initializes sidebar:
<details>
  <summary>Show me the code</summary>
    
```csharp

    protected virtual Sidebar GetSidebar()
    {
      var result = new Sitecore.Shell.Applications.ContentManager.Sidebars.Tree();
      result.ID = "Tree";
      result.DataContext = new DataContext()
      {
        DataViewName = "Master"
      };
      return (Sidebar) Assert.ResultNotNull<Sitecore.Shell.Applications.ContentManager.Sidebars.Tree>(result);
    }

```
</details>

Second, executed during context change (i.e. new item selection):
```csharp
    protected void OnDataContextChanged(Message message)
    {
      Assert.ArgumentNotNull((object) message, nameof (message));
      Sidebar sidebar = this.Sidebar;
      if (sidebar == null || !sidebar.OnDataContextChanged(this.ContentEditorDataContext, message))
        return;
      Item folder = this.ContentEditorDataContext.GetFolder();
      if (folder == null || this.HasPendingUpdate && !(this.PendingUpdateItemUri == (ItemUri) null) && !(this.PendingUpdateItemUri.ItemID == folder.ID))
        return;
      this.PendingUpdateItemUri = folder.Uri;
      this.HasPendingUpdate = true;
    }
```

As we can see below, `OnDataContextChanged` method triggers very similar one on Sidebar/Content Tree object:
```csharp
    public override bool OnDataContextChanged(DataContext context, Message message)
    {
      Assert.ArgumentNotNull((object) context, nameof (context));
      Assert.ArgumentNotNull((object) message, nameof (message));
      string queryString = WebUtil.GetQueryString("ro");
      if (string.IsNullOrEmpty(queryString))
        return true;
      Item folder = context.GetFolder();
      if (folder == null)
        return true;
      Item obj = folder.Database.GetItem(queryString);
      if (obj == null || obj.ID == folder.ID || obj.Axes.IsAncestorOf(folder))
        return true;
      context.SetRootAndFolder(obj.ID.ToString(), obj.Uri);
      return false;
    }
```

Bingo - it's `public`, `virtual`, and we have access there to currently selected item (`this.FolderItem`) and to new one (`context.GetFolder()`) - perfect candidate for our feature implementation.

Time to do some coding - we need to achieve three things:
- Create own version of Content Tree implementation, which will be aware about site language.
- Use this implementation in our custom code for `ContentEditorForm`
- Update `Default.aspx` CodeBeside to use proper class

Pretty simple, isn't it?

Here's mine implementation of `LanguageSwitchingTree`:
```csharp
    public class LanguageSwitchingTree : Sitecore.Shell.Applications.ContentManager.Sidebars.Tree
    {
        public override bool OnDataContextChanged(DataContext context, Message message)
        {
            var currentSite = GetSite(FolderItem);
            var newSite = GetSite(context.GetFolder());

            if (newSite != null && currentSite != newSite)
            {
                var newLanguage = GetLanguageForSite(newSite);

                if (context.Language != newLanguage)
                {
                    Log.Info($"Changing content language from '{context.Language.Name}' to '{newLanguage.Name}.'", this);
                    context.Language = newLanguage;
                }
            }

            return base.OnDataContextChanged(context, message);
        }

        private static SiteInfo GetSite(Item item)
        {
            if (item == null)
            {
                return null;
            }

            var siteInfoList = Sitecore.Configuration.Factory.GetSiteInfoList().OrderByDescending(x => x.RootPath.Length);

            return siteInfoList.FirstOrDefault(x => item.Paths.FullPath.StartsWith(x.RootPath));
        }

        private static Language GetLanguageForSite(SiteInfo site)
        {
            var languageCode = site.ContentLanguage.OrIfEmpty(site.Language);
            Language parsedLanguage;

            if (languageCode.IsEmptyOrNull() || !Language.TryParse(languageCode, out parsedLanguage))
            {
                return Sitecore.Context.ContentLanguage;
            }

            return parsedLanguage;
        }
    }
```

In a short summary - we are checking, if new item belongs to other site than the current item. If yes, we are trying to determine which language given site uses, by checking `ContenLanguage` or `Language` if first one is empty. If language is not set at all, we are doing fallback to the `Sitecore.Context.ContentLanguage`. If item belongs to other site we need to set new language in the context. Very important notice - **we should change language only if it's different from the current one.** Without this check we will end within endless loop and we don't want to be in such situation. Here's implementation of `Language` property setter in `DataContext` class:
```csharp
    public Language Language
    {
      (...)
      set
      {
        Assert.ArgumentNotNull((object) value, nameof (value));
        this.SetViewStateString(nameof (Language), value.Name);
        this.ClearCurrentCache();
        this.OnChanged();
      }
    }
```
As you can see, `OnChanged()` is called always, even if language was changed to the same one, so all event/message handlers will be called one more time and one of those handlers, will finally call ours `LanguageSwitchingTree` which will set language to the same one and so on...

So far so good, so now we need to use our `LanguageSwitchingTree` in `ContentEditorForm`. To do it, we need it implement our own version of `GetSidebar()` method:
```csharp
    public class CustomContentEditorForm : Sitecore.Shell.Applications.ContentManager.ContentEditorForm
    {
        protected override Sidebar GetSidebar()
        {
            var result = new LanguageSwitchingTree
            {
                ID = "Tree",
                DataContext = new DataContext
                {
                    DataViewName = "Master"
                }
            };

            return Assert.ResultNotNull(result);
        }
    }
```

Last step - we need to modify `Default.aspx` and use `CustomContentEditorForm` as a CodeBeside:
```xml
<sc:CodeBeside runat="server" Type="Sitecore.ContentLanguage.Switcher.CustomContentEditorForm, Sitecore.ContentLanguage.Switcher" />
```

And that's it - we can enjoy our new Content Manager improvement:
![ezgif-7-b756595bf1f1](https://user-images.githubusercontent.com/1537372/71258902-8a60e200-2337-11ea-8694-480fa9b84c3f.gif)
