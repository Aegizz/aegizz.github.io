# Trading Game Websockets

Look if you know me personally you know I hate one thing, **losing**.

Now, when all my SWE friends enter trading comps hosted by High Frequency Trading companies, I get pretty disappointed. Not only do they beat me (pretty badly at that), they get awesome prizes too! Now in all seriousness this is fine, my friends are insanely talented, and deserve any winnings they get in these competitions. 

But here is a story about how, during career week at University I hacked (more cheated) a company's trading game.

Initially, I just gave it my best shot, no mischief (it wasn't great). But as a long shot, I decided to watch the developer tools as I played. You know, for *science*.

At first, I couldn't find anything. Nada. Zip. Well, what the hell? That doesn't make any sense.

See, that is *really* odd. The trading game had random questions coming through and signals that would pop up to indicate the market.

If that is really how it works, I *must* be able to find that data.

This is it. I've got no work due, a perfect opportunity. **Lets hack it.**


<img src="../vulns/Pasted image 20250901000904.png" style="display: block;
  margin-left: auto;
  margin-right: auto;
  width: 75%;">

Well. Now I feel dumb. There is data being sent. It's using websockets. 

You see that image above? Its sending **everything** over websockets, including signals and game answers. This is **wayyy** too easy.

Lets make a script to automate it.

```python
import asyncio
import websockets
import json

URL = "wss://<censored>/ws/e65ed16e-1042-4aac-8327-e6f972d120d5"
PLAYER_ID = "50cc97f7-e061-519e-862d-25c882cab50b"

connection_message = {
    "event": "connection",
    "player_id": "",
    "data": {
        "alias": "Aegizz",
        "player_id": PLAYER_ID,
        "token": ""
    }
}

start_message = {
    "event": "start",
    "player_id": "",
    "data": {
        "player_id": PLAYER_ID
    }
}

skip_message = {
    "event": "skip",
    "player_id": "",
    "data": {}
}

# This function will handle the puzzle impact and adjust trading accordingly
def handle_puzzle_impact(puzzle_data):
    impact = puzzle_data.get("impact", 0)
    if impact > 0:
        print(f"The stock will increase by ${impact}.")
        return impact  # Indicating stock will increase (buy)
    elif impact < 0:
        print(f"The stock will decrease by ${abs(impact)}.")
        return impact  # Indicating stock will decrease (sell)
    return 0  # No trade if no clear impact

async def connect():
    async with websockets.connect(URL) as websocket:
        print("Connected to WebSocket")
        await websocket.send(json.dumps(connection_message))
        print("Sent connection message")

        while True:
            try:
                response = await asyncio.wait_for(websocket.recv(), timeout=5)
                print(f"Received: {response}")
                response_data = json.loads(response)

                if response_data.get("event") == "connection" and "data" in response_data:
                    if response_data["data"].get("player_id") == PLAYER_ID:
                        print("Connection established, sending start event...")
                        await websocket.send(json.dumps(start_message))
                        print("Sent start message")

                elif response_data.get("event") == "state" and "data" in response_data:
                    forecast = response_data["data"].get("price_forecast", 0)
                    position = response_data["data"].get("position", 0)
                    position_limit = response_data["data"].get("position_limit", 3)

                    if forecast > 0 and position < position_limit:
                        trade_volume = 3
                    elif forecast < 0 and position > -position_limit:
                        trade_volume = -3
                    else:
                        trade_volume = 0

                    if trade_volume != 0:
                        trade_message = {
                            "event": "trade",
                            "player_id": PLAYER_ID,
                            "data": {
                                "volume": trade_volume
                            }
                        }
                        await websocket.send(json.dumps(trade_message))
                        print(f"Sent trade: {'BUY' if trade_volume > 0 else 'SELL'} {abs(trade_volume)}")
                elif response_data.get("event") == "end" and "data" in response_data:
                    print("Game over!")
                    break  # Exit the loop when the game ends

                # Handling the puzzle event
                elif response_data.get("event") == "puzzle" and "data" in response_data:
                    puzzle_data = response_data["data"]
                    puzzle_impact = handle_puzzle_impact(puzzle_data)

                    # If puzzle impact is positive (stock increases), buy more stock
                    if puzzle_impact > 0:
                        trade_message = {
                            "event": "trade",
                            "player_id": PLAYER_ID,
                            "data": {
                                "volume": 3  # Buy more stock
                            }
                        }
                        await websocket.send(json.dumps(trade_message))
                        print("Sent trade: BUY 3")

                    # If puzzle impact is negative (stock decreases), sell stock
                    elif puzzle_impact < 0:
                        trade_message = {
                            "event": "trade",
                            "player_id": PLAYER_ID,
                            "data": {
                                "volume": -3  # Sell stock
                            }
                        }
                        await websocket.send(json.dumps(trade_message))
                        print("Sent trade: SELL 3")

                    # After the trade, send the skip message to move to the next puzzle/event
                    await websocket.send(json.dumps(skip_message))
                    print("Sent skip message")

            except asyncio.TimeoutError:
                continue
            except Exception as e:
                print(f"Error: {e}")
                break


def main():
    asyncio.run(connect())

if __name__ == "__main__":
    main()
```

Oh yeah! Wooooo hooo! Top of the leaderboard. **Huge**.


<img src="../vulns/Pasted image 20250901000459.png" style="display: block;
  margin-left: auto;
  margin-right: auto;
  width: 75%;">

In all seriousness that is a glaring security issue, and I'm surprised more people didn't exploit it.

After responsibly reporting it to the organisers, I sadly got no prize which is fair. Frustratingly though, they also didn't give a prize to any of the other contestants. Which is a massive shame since the all players below me, won fairly. 






