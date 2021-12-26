---
title: "Leafblower: Cards Against Humanity Clone Using Elixir"
date: 2021-12-25T17:01:06-06:00
draft: false
---

When the pandemic started me and other friends are looking for a game to play online. One of the games we found was https://pyx-1.pretendyoure.xyz/zy/game.jsp which is a Cards Against Humanity clone written in Java. They did a really good job of implementing features like being able to pick card packs you want & in-game chat feature. The problem is that it's not mobile friendly so you'll have to play it through the desktop view. 


I've been using this programming language called Elixir at my job and figured that it might be a good idea to write a CAH clone using it. So i created [Leafblower](https://leafblower.fly.dev/)

![Player picking a card](/leafblower-cah-clone/leafblower_card_pick.png)

It has the following features

- In-game chat
- It's mobile friendly so you can play it on the go
- It currently uses the base CAH card pack

# Show me the code!

It all starts by generating a game code. We call `GameStatem.generate_game_code/1` to find an unused game code first. Then we call `GameSupervisor.new_game` to start a game

[leafblower_web/controllers/game_splash_live.ex](https://github.com/fmterrorf/leafblower/blob/main/lib/leafblower_web/controllers/game_splash_live.ex#L51)

```elixir
  def handle_event("new_game", %{"user" => params}, socket) do
    {:ok, code} = Leafblower.GameStatem.generate_game_code()

    data =
      cast_user(params)
      |> Ecto.Changeset.apply_changes()

    {:ok, game} =
      Leafblower.GameSupervisor.new_game(id: code, countdown_duration: 120, min_player_count: 2)

    Leafblower.GameStatem.join_player(game, socket.assigns.user_id, data.name)

    {:noreply,
     socket
     |> push_redirect(to: Routes.live_path(socket, LeafblowerWeb.GameLive, code), replace: true)}
  end
```

When the `GameSupervisor.new_game/1` is called it will start 2 process: 

1. GameTicker - The process that handles countdown timers and publishes a tick message everytime it ticks
2. GameStatem - This stores the current state of the game as well as additional data like player names and ids

```elixir
 def new_game(arg) do
    {:ok, _} =
      Horde.DynamicSupervisor.start_child(
        __MODULE__,
        {GameTicker, Keyword.take(arg, [:id])}
      )

    Horde.DynamicSupervisor.start_child(
      __MODULE__,
      {GameStatem, arg}
    )
  end
```


The [GameStatem](https://github.com/fmterrorf/leafblower/blob/main/lib/leafblower/game_statem.ex#L1-L28) is where most of the magic happens. If you look at the file, you'll see the `status` and `data` typesecs

```elixir
defmodule Leafblower.GameStatem do
  @type player_id :: binary()
  @type player_info :: %{player_id() => %{name: binary()}}
  @type status :: :waiting_for_players | :round_started_waiting_for_response | :round_ended
  @type data :: %{
          id: binary(),
          active_players: MapSet.t(player_id()),
          player_info: player_info(),
          round_number: non_neg_integer(),
          round_player_answers: %{player_id() => list(binary())},
          required_white_cards_count: non_neg_integer | nil,
          leader_player_id: player_id() | nil,
          winner_player_id: player_id() | nil,
	  ....
        }
```


When the game is started, the default `status` will be `:waiting_for_players`. This will change as the game progresses. The controller sends messages to the `GameStatem` process and it will broadcast any changes to its state and data through an internal handle_event callback


```elixir
  def handle_event(:internal, :broadcast, status, data) do
    Phoenix.PubSub.broadcast(
      Leafblower.PubSub,
      topic(data.id),
      {:game_state_changed, status, Map.drop(data, [:deck])}
    )

    :keep_state_and_data
  end
```

# Closing thoughts

I had extreme fun building this game. Getting feedback from my friends who played was also very helpful. I hope you enjoy as well!
