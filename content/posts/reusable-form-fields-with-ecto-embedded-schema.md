---
title: "Reusable Form Fields With Ecto Embedded Schema"
date: 2022-03-19T19:34:09-06:00
draft: false
---

## Motivation

While working with forms in LiveView I ran into a situation where i want to re-use section of forms. Take the following forms for example:

* ![All fields](/reusable-forms/all-fields.png)

* ![Email Pref](/reusable-forms/email-pref.png)


Notice that the `Email Preferences` section is being used in two forms. 


## Enter Nested Embedded Schema

Let's create an component that contains a `Form` our form schema. 


```Elixir
defmodule ReusableFormsEmbeddedschemaWeb.EmailPreferencesComponent do
  use ReusableFormsEmbeddedschemaWeb, :component

  defmodule Form do
    use Ecto.Schema
    import Ecto.Changeset

    embedded_schema do
      field :alerts, :boolean
      field :product_updates, :boolean
    end

    def changeset(form, params \\ %{}) do
      form
      |> cast(params, [:alerts, :product_updates])
      |> validate_required([:alerts, :product_updates])
    end
  end

  def form_fields(assigns) do
    ~H"""
     <div>
        <div>
          <%= label @form, :alerts %>
          <%= checkbox @form, :alerts %>
          <%= error_tag @form, :alerts %>
        </div>
        <div>
          <%= label @form, :product_updates %>
          <%= checkbox @form, :product_updates %>
          <%= error_tag @form, :product_updates %>
        </div>
    </div>
    """
  end
end
```

The `ReusableFormsEmbeddedschemaWeb.EmailPreferencesComponent.Form` submodule uses an embedded schema. We also have a `changset/2` function that casts external data and adds validations for both the `:alerts` and `:product_updates`. Since this is a `Phoenix.Component`, we can we use it in other places by doing

```html
<EmailPreferencesComponent.form_fields form={@form} />
```

Notice how it accepts a `form`. That is infact a [Phoenix.HTML.Form](https://hexdocs.pm/phoenix_html/Phoenix.HTML.Form.html) and we'll how it's being passed later. 

### Making use of our new Component

Below is roughly how we can build the form from the first screenshot above

```Elixir
defmodule ReusableFormsEmbeddedschemaWeb.PageLive do
  use ReusableFormsEmbeddedschemaWeb, :live_view
  alias ReusableFormsEmbeddedschemaWeb.EmailPreferencesComponent

  defmodule Form do
    use Ecto.Schema
    import Ecto.Changeset

    embedded_schema do
      field :first_name, :string
      field :last_name, :string
      embeds_one :email_preferences, EmailPreferencesComponent.Form
    end

    def changeset(form, params \\ %{}) do
      form
      |> cast(params, [:first_name, :last_name])
      |> cast_embed(:email_preferences, with: &EmailPreferencesComponent.Form.changeset/2)
      |> validate_required([:first_name, :last_name])
    end
  end

  @impl true
  def mount(_params, _session, socket) do
    # You could read the DB to populate these fields
    form = %Form{
      first_name: nil,
      last_name: nil,
      email_preferences: %EmailPreferencesComponent.Form{
        alerts: false,
        product_updates: false
      }
    }

    changeset = Form.changeset(form)

    {:ok, assign(socket, changeset: changeset, form: form)}
  end

  @impl true
  def handle_event("validate", %{"form" => params}, socket) do
    changeset =
      socket.assigns.form
      |> Form.changeset(params)
      |> Map.put(:action, :insert)

    {:noreply, assign(socket, changeset: changeset)}
  end

  @impl true
  def handle_event("submit", %{"form" => params}, socket) do
    changeset =
      socket.assigns.form
      |> Form.changeset(params)

    form = Ecto.Changeset.apply_changes(changeset)
    # Save the data we care about somewhere

    changeset = Form.changeset(form)

    socket = put_flash(socket, :info, "Form Saved")

    {:noreply, assign(socket, changeset: changeset, form: form)}
  end

  @impl true
  def render(assigns) do
    ~H"""
    <div>
      <.form let={f} for={@changeset} phx-change="validate" phx-submit="submit">
        <div>
          <%= label f, :first_name %>
          <%= text_input f, :first_name %>
          <%= error_tag f, :first_name %>
        </div>
        <div>
          <%= label f, :last_name %>
          <%= text_input f, :last_name %>
          <%= error_tag f, :last_name %>
        </div>
        <div>
          <%= label f, :email_preferences %>
          <%= for email_pref_form <- inputs_for(f, :email_preferences) do %>
            <EmailPreferencesComponent.form_fields form={email_pref_form}/>
          <% end %>
        </div>

        <%= submit "Save", disabled: !@changeset.valid? %>
      </.form>
    </div>
    """
  end
end
```


Let's start unpacking the important parts. In our `Form` submodule, I used the `embeds_one` funciton to embed the `EmailPreferencesComponent.Form`. 


```Elixir
embedded_schema do
    field :first_name, :string
    field :last_name, :string
    embeds_one :email_preferences, EmailPreferencesComponent.Form
end

def changeset(form, params \\ %{}) do
    form
    |> cast(params, [:first_name, :last_name])
    |> cast_embed(:email_preferences, with: &EmailPreferencesComponent.Form.changeset/2)
    |> validate_required([:first_name, :last_name])
end
```


The most important part is this piece of code

```html
<div>
    <%= label f, :email_preferences %>
    <%= for email_pref_form <- inputs_for(f, :email_preferences) do %>
        <EmailPreferencesComponent.form_fields form={email_pref_form}/>
    <% end %>
</div>
```

We used [inputs_for/2](https://hexdocs.pm/phoenix_html/Phoenix.HTML.Form.html#inputs_for/2) here to get the form for `:email_preferences`. Since it returns a list, we need to loop using list comprehension, but we're sure that there's always only 1 item in that list since we're using `embeds_one`. 


Now you can use `EmailPreferencesComponent.form_fields/1` anywhere you want as long as you pass the required `form` property

Here's a demo of how it looks like


{{< rawhtml >}} 

<video width=100% controls autoplay>
    <source src="/reusable-forms/final.webm" type="video/webm">
    Your browser does not support the video tag.  
</video>

{{< /rawhtml >}}
