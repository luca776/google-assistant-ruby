# Google Assistant Ruby

Write Google Assistant actions in Ruby.

GoogleAssistant parses Google Assistant requests and provides the framework to respond appropriately. It works with the Google Assistant API v1.

## Installation

Add this line to your application's Gemfile:

```rb
gem "google_assistant"
```

## Get started

### Rails 4 and higher

In your routes file, allow a `POST` request to your action. All requests from Google Assistant will come here.

```rb
Rails.application.routes.draw do
  post "/google_assistant" => "google_assistant#conversation"
end
```

Write your controller to handle the request. Use GoogleAssistant to respond to the requests.

```rb
class GoogleAssistantController < ApplicationController

  def conversation
    assistant_response = GoogleAssistant.respond_to(params, response) do |assistant|
      assistant.intent.main do
        assistant.ask(
          prompt: "<speak>Hi there! Say something, please.</speak>",
          no_input_prompt: [
            "<speak>If you said something, I didn't hear you.</speak>",
            "<speak>Did you say something?</speak>"
          ]
        )
      end

      assistant.intent.text do
        assistant.tell("<speak>I can respond, too!</speak>")
      end
    end

    render json: assistant_response
  end
end
```

## Usage

GoogleAssistant parses the request from the Google Assistant API and helps you build your response. It takes the `params` and `response` objects in Rails and Sinatra.

```rb
assistant_response = GoogleAssistant.respond_to(params, response) do |assistant|
  # Response logic goes here
end
```

The Google Assistant API sends a request using the `MAIN` intent for the initial request from the user. This is when the user says "OK Google, talk to #{name_of_your_app}".

```rb
assistant_response = GoogleAssistant.respond_to(params, response) do |assistant|
  assistant.intent.main do
    # Initial request's response goes here. You may want to introduce the app here.
  end
end
```

The Google Assistant API sends a request using the `TEXT` intent for subsequent requests from the user. When your app asks for input from the user, the conversation continues. When the user finishes the conversation with "Goodbye" or your app finishes the conversation with a `tell` response, the conversation ends and any new conversations start again with the `MAIN` intent.

```rb
assistant_response = GoogleAssistant.respond_to(params, response) do |assistant|
  assistant.intent.text do
    # Respond to user input here.
  end
end
```

### Ask

Request user input by sending an `ask` response. Send a prompt and a set of follow-up prompts for when the user fails to respond.

```rb
assistant.intent.main do
  assistant.ask(
    prompt: "<speak>Hi there! Say something, please.</speak>",    # The voice prompt the user will hear.
    no_input_prompt: [
      "<speak>If you said something, I didn't hear you.</speak>", # You can provide a number of "no input prompts". A random
      "<speak>Did you say something?</speak>"                     # one will be spoken if the user takes too long to respond.
    ]
  )
end
```

### Tell

Send a final response, ending the conversation.

```rb
assistant.intent.text do
  assistant.tell("<speak>Thanks for talking! Goodbye!</speak>")   # Both SSML and plain text work here and anywhere you send a prompt.
end
```

### SSML

SSML is Google Assistant's markup language for text to speech. It provides options to pause, interpret dates and numbers, and more. You can provide SSML responses or plain text. See [Google's documentation on SSML](https://developers.google.com/actions/reference/ssml).

### User input

GoogleAssistant allows you to read the user's input using `assistant.arguments` so that you can respond appropriately.

```rb
assistant.intent.text do
  case assistant.arguments[0].text_value.downcase
  when "hello"
    respond_with = "Hi there!"
  when "goodbye"
    respond_with = "See you later!"
  else
    respond_with "I heard you say #{assistant.arguments[0].text_value}, but I don't know what that means."
  end

  assistant.tell(respond_with)
end
```

### Conversation state and data

You can keep data on the Conversation object. This data will be sent to the Google Assistant API, which will return that data back when the user responds. This way, you can keep ongoing information about the conversation without having to use a database to store that information.

You can also send a state value with your responses to keep track of where your conversation is at.

```rb
GoogleAssistant.respond_to(params, response) do |assistant|
  assistant.intent.main do
    assistant.conversation.state = "asking favorite color"

    assistant.ask(
      prompt: "What is your favorite color?",
      no_input_prompt: ["What did you say your favorite color is?"]
    )
  end

  assistant.intent.text do
    if assistant.conversation.state == "asking favorite color"
      assistant.conversation.data["favorite_color"] = assistant.arguments[0].text_value

      assistant.conversation.state = "asking lucky number"

      assistant.ask(
        prompt: "What is your lucky number?",
        no_input_prompt: ["What did you say your lucky number is?"]
      )
    elsif assistant.conversation.state == "asking lucky number"
      favorite_color = assistant.conversation.data["favorite_color"]
      lucky_number = assistant.arguments[0].text_value

      assistant.tell("Your favorite color is #{favorite_color} and your lucky number is #{lucky_number}. Neat!")
    end
  end
end
```

### User ID

You can get the user's ID. This will allow you to identify the user across conversations. It works much in the same way a cookie might work; it is possible for the user to reset that ID, so don't rely on it too much.

```rb
# Get the user's ID
assistant.user.id
```

### Permissions

You can request information about the user and their device. Google handles collecting this information, but you can provide a prompt to let the user know why you need this information. The Google Assistant API responds with the `permission` intent after a permission request.

#### Name

Request the user's name. This will result in a prompt to the user like:
> So that I can address you by name, I'll just need to get your name from Google. Is that ok?

```rb
assistant.intent.main do
  # Request the user's name
  assistant.ask_for_permission(context: "So that I can address you by name", permissions: GoogleAssistant::Permission::NAME)
end

assistant.intent.permission do
  if assistant.permission_granted?
    # Get the user's name from the response
    given_name = assistant.user.given_name
    family_name = assistant.user.family_name
    display_name = assistant.user.display_name
  else
    # The user denied permission
  end
end
```

#### Coarse Location

Request the device's zip code and city. This will result in a prompt to the user like:
> To provide weather information for where you live, I'll just need to get your zip code from Google. Is that ok?

```rb
assistant.intent.main do
  # Request the device's zip code and city
  assistant.ask_for_permission(context: "To provide weather information for where you live", permissions: GoogleAssistant::Permission::DEVICE_COARSE_LOCATION)
end

assistant.intent.permission do
  if assistant.permission_granted?
    # Get the device's location from the response
    zip_code = assistant.device.zip_code
    city = assistant.device.city
  else
    # The user denied permission
  end
end
```

#### Precise Location

Request the device's precise location. This will result in a prompt to the user like:
> So that I can find out where you sleep at night, I'll just need to get your street address from Google. Is that ok?

```rb
assistant.intent.main do
  # Request the device's precise location
  assistant.ask_for_permission(context: "So that I can find out where you sleep at night", permissions: GoogleAssistant::Permission::DEVICE_PRECISE_LOCATION)
end

assistant.intent.permission do
  if assistant.permission_granted?
    # Get the device's location from the response
    zip_code = assistant.device.zip_code
    city = assistant.device.city
    formatted_address = assistant.device.formatted_address
    latitude = assistant.device.latitude
    longitude = assistant.device.longitude
  else
    # The user denied permission
  end
end
```

### Testing your assistant

You can use any hosting platform.

1. [Download the `gactions` CLI](https://developers.google.com/actions/tools/gactions-cli) and add it to your PATH.
    - Or if you'd rather not put it in your path, you'll simply need to call it by referencing its full path.
2. Visit the [Google Cloud Console projects page](https://console.cloud.google.com/project). Create a project and make note of the project ID for configuration and deployment.
3. Deploy your app to the web. Heroku is a good choice. See [Heroku's documentation](https://devcenter.heroku.com/articles/getting-started-with-ruby#introduction) for more info on how to do this.
4. Add an `action.json` file at the root of your project.

    ```json
    {
      "versionLabel": "1.0.0",
      "agentInfo": {
        "languageCode": "en-US",
        "projectId": "your-google-project-id",
        "voiceName": "male_1"
      },
      "actions": [
        {
          "initialTrigger": {
            "intent": "assistant.intent.action.MAIN"
          },
          "httpExecution": {
            "url": "https://yourapp.domain.com/path-to-your-assistant"
          }
        }
      ]
    }
    ```

5. Run the following command from the root directory of your project. The `invocation_name` is what you will use to activate your assistant. For example, "OK Google, talk to my action". Name it something unique.

    ```
    gactions preview -action_package=action.json -invocation_name="my action"
    ```

    - `gactions` will ask to access your account and Google developer project. Follow the onscreen instructions to do so.

6. Use the [web simulator](https://developers.google.com/actions/tools/web-simulator) to simulate. Or better yet, if your Google Home device is logged into the same account you're using to build your action, you can say "OK Google, talk to my action" to test it out directly on your device.

## Authentication

You can require users to log in to your service before using your assistant. Read about it in [Google's documentation](https://developers.google.com/actions/develop/identity/oauth2-code-flow). The basic flow is this:

1. User tries to talk to your assistant
2. Google tells the user they need to sign in, which they can do via the Home app on their phone
3. The Home app links the user to your Oauth endpoint
4. User signs in to your app
5. Google stores the user's Oauth access and refresh tokens
6. For each subsequent request the user makes to your assistant, Google sends the user's access token so you can identify the user

In order to set this up in your assistant, the basic instructions are as follows. Read Google's documentation for the full details.

1. Implement Oauth in your application
2. Set up an Oauth client in the [Google Developer Console](https://console.developers.google.com)
3. In the application's `action.json` file, set up account linking according to [Google's Instructions](https://developers.google.com/actions/develop/identity/account-linking#enabling_account_linking)
4. Use `assistant.user.access_token` to identify the user

## More information

Check out Google's instructions at https://developers.google.com/actions/develop/sdk/getting-started for more detail on writing and testing a Google Assistant action.

Check out https://github.com/armilam/google_assistant_example for a simple example of this gem in action.
