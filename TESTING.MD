# Blackjack Game Testing Report

```
This `testing.md` file outlines the key issues, the solutions implemented, and provides code snippets where necessary. It can serve as a living document to track bugs and improvements in the game. Let me know if you need further details or more sections!
````

## Table of Contents
1. [Bug: Session Issues After Logout](#bug-session-issues-after-logout)
2. [Bug: Player Balance Resets Incorrectly](#bug-player-balance-resets-incorrectly)
3. [Bug: Doubled Winnings on Leaderboard](#bug-doubled-winnings-on-leaderboard)
4. [Bug: Flash Messages Positioning](#bug-flash-messages-positioning)
5. [Bug: Game Not Resetting on Logout](#bug-game-not-resetting-on-logout)
6. [Bug: Hit and Stand Buttons Active After Logout](#bug-hit-and-stand-buttons-active-after-logout)
7. [Bug: Player Allowed to Place a Bet Greater than Balance](#bug-player-allowed-to-place-a-bet-greater-than-balance)
8. [Bug: Login Signup Issues](#bug-login-signup-issues)
9. [Bug: Card Count Not Updating Correctly](#bug-card-count-not-updating-correctly)
10. [Bug: Incorrect Ace Value Adjustment](#bug-incorrect-ace-value-adjustment)
11. [Bug: Incorrect Card Representation](#bug-incorrect-card-representation)
12. [Bug: Premature Deck Depletion](#bug-premature-deck-depletion)
13. [Bug: Currency Balance Not Resetting to £1000 When User Balance Reaches £0](#bug-currency-balance-not-resetting-to-£1000-when-user-balance-reaches-£0)



---

## Bug: Session Issues After Logout

### Description:
When logging out, the session data for the current game is not cleared, and a player may continue a previous game after re-logging.

### Fix:
In the `/logout` route, added logic to clear session data for `game_in_progress`, `current_bet`, and `current_balance` to ensure the game state resets after logout.

### Code:
```python
@app.route("/logout", methods=["POST"])
@login_required
def logout():
    if 'game_in_progress' in session:
        game.reset_game()

    # Clear session data
    session.pop('game_in_progress', None)
    session.pop('current_bet', None)
    session.pop('current_balance', None)

    logout_user()  # Log out the user
    flash('You have logged out successfully.')
    return redirect(url_for('home_route'))
```

### Comments
Clearing session data upon logout ensures that players cannot continue an interrupted game after re-logging in.

## Bug: Player Balance Resets Incorrectly

### Description:
In the reset_game() method, check the player's current balance from MongoDB and only reset to £1000 if the balance is £0. Otherwise, carry forward the current balance.

### Fix:
In the reset_game() method, check the player's current balance from MongoDB and only reset to £1000 if the balance is £0. Otherwise, carry forward the current balance.

### Code:
```python
def reset_game(self):
    current_balance = db.users.find_one({"username": current_user.username})["balance"]
    if current_balance == 0:
        self.balance = 1000  # Reset only if balance is zero
    else:
        self.balance = current_balance  # Carry forward existing balance
```       

### Comments
This ensures that players don’t lose their progress unless they reach a game-over state.

## Bug: Doubled Winnings on Leaderboard

### Description:
The winnings were being added to the leaderboard twice: once for the current bet and once for the total winnings.

### Fix:
Adjusted the way self.winnings is passed to self.update_leaderboard, ensuring only the correct amount is recorded.

### Code:

```python
 self.update_leaderboard(player_name=self.player_name, winnings=self.winnings)
```

### Comments
This fix ensures that winnings are only recorded once on the leaderboard, reflecting the correct total.

## Bug: Flash Messages Positioning

### Description:
Flash messages appear at the top of the screen and are hard to notice. They should be styled and centered on the page.

### Fix:
Custom CSS was added to center align the flash messages, make them bold, and style them with specific colors.

### Code:
```CSS
.flash-message {
    text-align: center;
    font-weight: bold;
    color: #f00; /* Red for error */
    display: inline-block;
    background-color: #ffebcc;
    padding: 10px;
    border-radius: 5px;
    margin: 20px auto;
}
```
### Comments
This improves user experience by making flash messages more prominent and aesthetically pleasing.

## Bug: Game Not Resetting on Logout

### Description:
When a player logs out mid-game, the game is not properly reset, allowing them to resume upon re-logging.

### Fix:
In the /logout route, the game.reset_game() function is called before clearing session data, ensuring the game state resets.

### Code:
```python
if 'game_in_progress' in session:
    game.reset_game()  # Resets game state including the player's balance
```

### Comment:
Now the game resets correctly when the user logs out, preventing any unintended continuation of the game.

## Bug: Hit and Stand Buttons Active After Logout

### Description:
The hit and stand buttons remain active on the game interface even after the player logs out.

### Fix:
Added logic to disable these buttons when current_user.is_authenticated is False.

### Code (Jinja2:
```html
{% if current_user.is_authenticated and game_state.is_game_active and not game_state.game_over %}
    <button id="hit-btn">Hit</button>
    <button id="stand-btn">Stand</button>
{% else %}
    <button id="hit-btn" disabled>Hit</button>
    <button id="stand-btn" disabled>Stand</button>
{% endif %}

```
## Bug: Player Allowed to Place a Bet Greater than Balance

### Description:
Players can place bets higher than their current balance, leading to negative balances.

### Fix:
Added a validation step in the set_bet method to check if the bet exceeds the available balance.

### Code:
```python
def set_bet(self, amount):
    if amount > self.balance:
        raise ValueError("Bet exceeds current balance.")
    self.balance -= amount  # Deduct the bet from the balance
```
## Bug: Login Signup Issues

### Description:
- Encountered error message: "An error occurred: Missing user_loader or request_loader. Refer to http://flask-login.readthedocs.io/#how-it-works for more info."
- This error occurs when Flask-Login cannot find a loader function to retrieve the logged-in user from the database.

### Fix:
 Flask-Login uses the user ID as a string, but MongoDB returns it as an ObjectId. Converting the _id to a string resolves the issue.
 
### Debugging: 
  - The issue was discovered by checking <p>Authenticated: {{ current_user.is_authenticated }}</p> and noticing it kept returning False.
- Fixed after referring to StackOverflow, which explained how to properly convert MongoDB   _id to a string to match Flask-Login’s requirements.

### Code:
```python
@login_manager.user_loader
def load_user(user_id):
    user_data = user_collection.find_one({"_id": user_id})
    if user_data:
        return User(str(user_data['_id']), user_data['username'], user_data['email'])
    return None
```

## Bug: Card Count Not Updating Correctly

### Description:
The count used for tracking card distribution wasn't updating properly when face cards are dealt.

### Fix:
Fixed the face cards (Jack, Queen, King) reducing the count as expected in Deck.update_count().

### Code:
```python
def update_count(self, card):
    if card.rank in ['2', '3', '4', '5', '6']:
        self.count += 1
    elif card.rank in ['10', 'Jack', 'Queen', 'King', 'Ace']:
        self.count -= 1

```
## Bug: Incorrect Ace Value Adjustment

### Description:
Aces are not being adjusted correctly, sometimes causing the player to bust prematurely.

### Fix:
Added code to check for Ace value adjustment more carefully after adding a card to the hand. Ensure that multiple Aces are handled properly by reducing the value of 11 to 1 when necessary.

### Code:
```python
def adjust_for_ace(self):
    while self.value > 21 and self.aces:
        self.value -= 10
        self.aces -= 1

```
## Bug: Incorrect Card Representation

### Description:
The cards in the game are not showing the correct images or displaying properly due to issues with the path generation logic.

### Fix:
Implemented the card_to_string() function to dynamically generate the image path based on the card's suit and rank. Adjustments were made to ensure that singular forms of suits are used in the image path.

### Code:

```python
def card_to_string(self, card):
    """Return the path to the card's image based on its suit and rank."""
    suit = card.suit.lower()  # 'hearts', 'diamonds', 'clubs', 'spades'
    rank = card.rank.lower()  # '2', '3', ..., '10', 'jack', 'queen', 'king', 'ace'
    suit_name = suit[:-1] if suit.endswith('s') else suit  # e.g., 'hearts' to 'heart'
    return f"/static/images/cards/card-{suit_name}-{rank}.png"

```

### Comments:
- Suit Handling: The suit names in the image path were pluralized (e.g., 'hearts'), but the image names require them to be singular (e.g., 'heart').
- Rank Handling: Ensures lowercased rank (e.g., '2', 'jack', 'ace') is used consistently.

## Bug: Premature Deck Depletion

### Description:
Cards are running out before the game finishes because the deck is not reshuffled or re-initialized after all cards are dealt.

### Fix:
Added a check in the deal() function to handle the case when the deck runs out of cards.

### Code:

```python
def deal(self):
    """Deal one card from the deck."""
    if not self.cards:
        raise Exception("No more cards in the deck.")
    card = self.cards.pop()
    self.update_count(card)  # Update count as cards are dealt
    return card

```
## Bug: Currency Balance Not Resetting to £1000 When User Balance Reaches £0

### Description:
- The user's balance does not reset to £1000 when the balance reaches £0, as expected. Instead, the game allows the balance to remain at £0, preventing the player from continuing the game.
- This issue arises because the condition for resetting the balance is not triggered properly when both balance and bet are 0.

### Expected Behavior:
- When the player's balance reaches £0 (after losing all available currency), the balance should reset to £1000 to allow the player to continue playing.

### Fix:
Added a conditional check to ensure that when both the balance and bet are 0, the balance is automatically reset to £1000.

### Code:
```python
if self.balance <= 0 and self.bet <= 0:
    self.balance = 1000
```   

### Comments:
- Reset Logic: The balance reset logic ensures that the user is able to continue playing when their balance is depleted. By checking both balance <= 0 and bet <= 0, the game ensures that the reset occurs only when the player has completely run out of funds and has no ongoing bets.
- Conditional Safeguard: The addition of self.bet <= 0 is a safeguard against resetting the balance while the user has an active bet, which might lead to unintended outcomes.

### Detailed Testing and Scenarios:

##### Scenario 1: Balance Depletes to £0
- Steps:
     1. Player places bets until their balance is reduced to £0.
     2. Verify if the balance automatically resets to £1000 after the next round starts.
     
- Expected Outcome:
    - Once the balance reaches £0 and the player has no active bets, the balance should reset to £1000.
    
- Comments:
     - Ensure that the reset happens before the next game starts to avoid blocking the game due to insufficient funds.
    
##### Scenario 2: Active Bet While Balance is £0

- Steps:
     1. Player places a bet that reduces the balance to £0 but has an active bet.
     2. Ensure that the balance does not reset during the active round.
     
- Expected Outcome:
      - The balance should not reset to £1000 while the player still has an active bet.

- Comments:
     - The reset should only happen when both the balance and bet are 0.

##### Scenario 3: Reset After Losing Final Bet

- Steps:
     1. Player places their last bet, reducing the balance to £0.
     2. The player loses the bet, so both balance and bet are now 0.
     3. Verify if the balance resets to £1000 after the round.
     
- Expected Outcome:
     - After the player loses their final bet, their balance should reset to £1000.

- Comments:
     - This scenario tests the condition self.balance <= 0 and self.bet <= 0 to ensure it triggers as expected.
     

    
    








