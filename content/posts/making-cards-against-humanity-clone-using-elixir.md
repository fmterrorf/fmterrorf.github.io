---
title: "Leafblower: Cards Against Humanity Clone Using Elixir"
date: 2021-12-25T17:01:06-06:00
draft: true
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




