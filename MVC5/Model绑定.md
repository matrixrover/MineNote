    namespace System.Web.Mvc {
        public interface IModelBinder {
            object BindModel(ControllerContext controllerContext,
                ModelBindingContext bindingContext);
        }
    }


<table>
    <caption>The Order in Which the <strong>DefaultModelBinder</strong> Class Looks for Parameter Data</caption>
    <thead>
        <tr>
            <th>Source</th>
            <th>Description</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>Request.Form</td>
            <td>Values provided by the user in HTML form elements</td>
        </tr>
        <tr>
            <td>RouteData.Values</td>
            <td>The values obtained using the application routes</td>
        </tr>
        <tr>
            <td>Request.QueryString</td>
            <td>Data included in the query string portion of the request URL</td>
        </tr>
        <tr>
            <td>Request.Files</td>
            <td>Files that have been uploaded as part of the request </td>
        </tr>
    </tbody>
</table>


When dealing with simple parameter types, the DefaultModelBinder tries to convert the string value, which has been
obtained from the request data into the parameter type using the **System.ComponentModel.TypeDescriptor** class.

the DefaultModelBinder class uses culture-specific settings to perform type conversions from different areas of
the request data. the values that are obtained from Urls (the routing and query string data) are converted using
culture-insensitive parsing, but values obtained from form data are converted taking culture into account.
the most common problem that this causes relates to **DateTime** values.

When the action method parameter is a complex type (i.e., any type which cannot be converted using the
TypeConverter class), then the DefaultModelBinder class uses reflection to obtain the set of public properties and
then binds to each of them in turn.

When the **Bind** attribute is applied to an action method parameter, it only affects instances of that class that are
bound for that action method; all other action methods will continue to try and bind all the properties defined by the
parameter type. If you want to create a more widespread effect, then you can apply the Bind attribute to the model
class itself.

When the Bind attribute is applied to the model class and to an action method parameter, a property will be
included in the bind process only if neither application of the attribute excludes it. this means that the policy applied to
the model class cannot be overridden by applying a less restrictive policy to the action method parameter.

---------------------------------

When manually invoke the binding process, the binding process can be restricted to a single source of data.


    UpdateModel(model, new FormValueProvider(ControllerContext));


<table>
    <caption>The Built-in IValueProvider Implementations</caption>
    <thead>
        <tr>
            <th>Source</th>
            <th>IValueProvider Implementation</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>Request.Form</td>
            <td>FormValueProvider</td>
        </tr>
        <tr>
            <td>RouteData.Values</td>
            <td>RouteDataValueProvider</td>
        </tr>
        <tr>
            <td>Request.QueryString</td>
            <td>QueryStringValueProvider</td>
        </tr>
        <tr>
            <td>Request.Files</td>
            <td>HttpFileCollectionValueProvider</td>
        </tr>
    </tbody>
</table>


The FormCollection class implements the IValueProvider interface, and if I define the action method to take
a parameter of this type, the model binder will provide me with an object that I can pass directly to the UpdateModel
method.

    public ActionResult Address(FormCollection formData) {
        IList<AddressSummary> addresses = new List<AddressSummary>();
        UpdateModel(addresses, formData);
        return View(addresses);
    }

there are other overloaded versions of the UpdateModel method that specify a prefix to search for and which
model properties should be included in the binding process.


    try {
        UpdateModel();
    } catch (InvalidOperationException ex) {
        // provide feedback to user
    }

    if (TryUpdateModel()) {
        // proceed as normal
    } else {
        // provide feedback to user
    }

The only reason to favor TryUpdateModel over UpdateModel is if you don’t like catching and dealing with
exceptions. There is no functional difference in the model binding process.

**When model binding is invoked automatically, binding errors are not signaled with exceptions. instead, you must
check the result through the ModelState.IsValid property.**

--------------------------------

# Customizing the Model Binding System

## Creating a Custom Value Provider

    namespace System.Web.Mvc {
        public interface IValueProvider {
            bool ContainsPrefix(string prefix);
            ValueProviderResult GetValue(string key);
        }
    }

The ContainsPrefix method is called by the model binder to determine if the value provider can resolve the data
for a given prefix. The GetValue method returns a value for a given data key, or null if the provider doesn’t have any
suitable data.

This **ValueProviderResult** class has three constructor parameters. The
first is the data item that I want to associate with the requested key. The second parameter is a version of the data
value that is safe to display as part of an HTML page. The final parameter is the culture information that relates to the
value; I have specified the InvariantCulture.

To register the value provider with the application, I need to create a factory class that will create instances of
the provider when they are required by the MVC Framework. The factory class must be derived from the abstract
**ValueProviderFactory** class.

    public class CustomValueProviderFactory : ValueProviderFactory {
        public override IValueProvider GetValueProvider(ControllerContext controllerContext) {
            return new CountryValueProvider();
        }
    }

The GetValueProvider method is called when the model binder wants to obtain values for the binding process.
This implementation simply creates and returns an instance of the CountryValueProvider class, but you can use
the data provided through the ControllerContext parameter to respond to different kinds of requests by creating
different value providers.

I need to register the factory class with the application, which I do in the Application_Start method of
Global.asax

    ValueProviderFactories.Factories.Insert(0, new CustomValueProviderFactory());

I register the factory class by adding an instance to the static ValueProviderFactories.Factories collection.
The model binder looks at the value providers in sequence, which means I have to use the Insert method to put the
custom factory at the first position in the collection if I want to take precedence over the built-in providers.

If I want the custom provider to be a fallback that is used when the other providers cannot supply a data value,
then I can use the Add method to append the factory class to the end of the collection, like this:

    ValueProviderFactories.Factories.Add(new CustomValueProviderFactory());

## Creating a Custom Model Binder