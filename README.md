# bd-exploit-info
### Explainations, and more detailed information about the exploit I have discovered over time.

**This writeup will also contain solutions to said exploits which seem reasonable/possible to do**

**A overview of exploits which I know of.**
- Using a trial-type character of any class outside of the battle-arena with limitations
- Near god mode, as well as high DPS output with a specific type of class
- Forcing the client to send out a single animation state, also known as animation-overflowing
- Teleportation using modification of the local position variables
- Teleportation using packet manipulation which also allows to bypass any speed limitation, height limitation
- Modification of the animation speeds which causes the client to process actions faster which in terms deals more damage, finish actions faster
- Instant gathering with some limitations, this applies to all types of gathering (butchering etc.)
- Flying using animation-underflowing for a specific class (I did not do enough research on other classes)
- Interacting with a NPC without having the part of the area loaded by the client, using packet manipulation
- Bypassing the indicator packet which is sent when a trial-type character exits the battle-arena or attempts such action - which also triggers the teleportation back into the battle-arena
- Locking the spawn-position permanently of any trial-type character to absolutely anywhere (no limitation and never changes even after server restarts)
- Unlimited underwater breathing
- Looting anything within range without any animation - instantly
- Bypassing pet looting cooldowns to loot up to 2x in one second for a T1 Lv1 pet (or any other)
- Bypassing the requirement of having full-hp and being idle in order of entering/exiting battle-arena

**-----**

**How to use a trial-type character outside of the battle-arena**

The way this works is quite simple; you need to be able to have good enough flying method to escape the battle arena, or use a teleporting based method.

The way you escape using a regular flying or cursor teleport method is, you fly up until you reach the border of the arena, once you reached it you can simply escape the arena if you have either the client setup to ignore the escape detection or use a bypass.

There are other types of methods as well which involve the savage rift, and the team battle (I don't actually remember the name of it).
For those the process is similar but involved bugging through objects such as walls.

The savage rift method; Once you enter the savage rift you are in the lobby, you can simply proceed to bug through the walls there to escape the lobby area, and do anything you like after that, such as grinding items or anything else for that matter.

The team battle method is similar to this, the only difference is that you need to escape through the ocean where a border is placed which is simple to bypass by bugging through it.

**Possible solution for this**
The height border for the battle-arena is not enough to prevent flying based escapes, the maximal height of said border could be increased to the absolute maximum, bugging through those borders is still possible even with this in place.

Position validation for trial-type character should also be taken into concideration to automatically detect escapes, and prevent them effectively, as well as disabling the access to the savage rift and team battle for trial-type charaters or restricting their position to those areas server sided.

**-----**

**How to achieve near god mode on the berserker class**

This exploit is used widely and I'm sure everyone is well aware of this; The way to achieve this is extremely simple, we use the class berserker and the skill "Beast Roar".

You activate the skill "Beats Roar" and then once you are in the stance "BT_skill_AggroShout_Ing_UP" you can set the current animation speed to a high value such as max int or any other value above 10.000 to achieve this effect.

The player will get stuck in the animation which triggers a attack, the server deosn't seem to exactly care about the extreme about of packets coming from the client (animation update type packets as well as another one which seemed to cause healing).

Moving is not possible in this state, this requires you to have a teleportation method which does not affect your current animation playing.

**Possible solution for this**
Limit the amount of animations possible for each stance type of skill and force a reset/disconnect the client if violated.

**-----**

**How to force the client to send out a single animation state**

As mentioned before this is using animation-overflowing and the same principal of "locking" a certain animation state.

There are not many classes which work with this method, any stance type skill can be affected by this method, the shai ultimate is affected by this exploit as well and can cause high damage, with the same limitation as usual - movement is not possible while in the "locked" state.

There are many ways to get reset in this state, the most common one is falling down or glitching through objects which will cause a animation change forcefully.

**-----**

**How to teleport using modification of the local position variables**

Here is a example of how to edit the position variable which is networked, the known limit is 4 units in one second.
```cpp
	auto _controller = *(uint64_t*)(_self + offset::actor::player_control); //0x3D8
	if (!_controller) return;
	auto _base = *(uint64_t*)(_controller + offset::actor::player_base);    //0x8
	if (!_base) return;
	auto _unk = *(uint64_t*)(_base + offset::actor::player_unk);            //0x198
	if (_unk != 0)
	{
		*(float*)(_unk + offset::actor::player_network_pos_x) = _pos.x;       //0x120
		*(float*)(_unk + offset::actor::player_network_pos_y) = _pos.y;       //0x128
		*(float*)(_unk + offset::actor::player_network_pos_z) = _pos.z;       //0x124
	}
```

The server seems to accept this if the distance is not too high from the current origin, and will cause rubber-banding once you attempt to teleport too often/fast.

I see no direct way of solving this with the way the movement system works in this game, so limiting this and validating it more strictly would be the best step to take.

**-----**

**How to teleport using packet manipulation**

This is particular is harder to pull of as it requires you to also have packet hooks to output in place, and some knowledge of how packets are constructed in this game.

Basic idea behind this type of teleportation is; you directly communicate to the server that a position update has occured, the server will perform such update on his side and update the client afterwards with a new position.

The server does still do checks to validate the position change, but those are way less intense and mostly go.
There is no real limit on how fast you can move as well with this so as long as you have line of sight to the target you can step your way to said target with a maximal step size of 8 to 10 units.

How to achieve this; You log the base packet which will be used to set a new position server-sided, the packet consists of a tick-count signature and around mentions of the wanted position (not sure why) and more other data which doesn't seem important for what we want to do.

Here is a example of how this would look like
```cpp
	auto packet = game::buffer_3811;
	/*fix tickcount*/
	packet.putLong(GetTickCount64(), game::timer_3811);
	/*set up x*/
	for (auto obj : game::pos_3811) packet.putFloat(_pos.x, obj);
	/*set up y*/
	for (auto obj : game::pos_3811) packet.putFloat(_pos.y, obj + 8);
	/*set up z*/
	for (auto obj : game::pos_3811) packet.putFloat(_pos.z, obj + 4);
	/*send pack*/
	game::send_packet(packet, game::get_next_id, packet.buf.size());
```

The array game::pos_3811 is a collection of the found positions in the packet data itself, which then is used to replace all positions to our target position.

The packet I used for this is triggered when the player jumps (because this is also a height modifier and is needed for other exploits using this).

**Possible solution for this**
A way to solve this from being abused is to verify the position for this packet more strictly, such as; calculating the maximal possible movement done in a certain time with the current walking speed or horse speed.

**-----**

**How to modify the animation speed**

Simple and well known, here is the code used to achieve this, not much is needed to be said about this as it simply speeds up any action performed by the player - most things such as alchemy etc. will not work with this due to sanity checks in place (luckily).

```cpp
  	auto _controll = *(uint64_t*)(self_player + offset::actor::player_control); //0x3D8
	if (!_controll) return;
	auto _scene = *(uint64_t*)(_controll + offset::actor::player_scene);        //0x10
  	*(float*)(_scene + offset::actor::player_animat) = wanted_speed;            //0x4C0
```

**Possible solution for this**
A way to solve this would be to check for the animation and how fast it should execute, if violated the user has a modified animation speed.

**-----**

**How to achieve instant gathering times**

This is a interesting conflict on the server-side, the way this works is you need a gathering speed of 6 seconds or lower.
As soon as you start gathering (butchering etc.) you can simple perform a call to (lua) 

```lua
getSelfPlayer():setActionChart('WAIT')
```

Which will result in the current action being reset, as well as finished on the server-side as the server just accepts you being done earlier than the timer says for no real reason.

**Possible solution for this**
A way to solve this would be to not just blindly accept this action, but have a timer on the server-side to see if the action actually should be done.

**-----**

**How to achieve flying using animation-underflowing for the berserker class**

This method uses the berserkers class awakening skill which is "Giant's Leap".
This enables us to fly around with some additional position adjustment such as height for better control.

The way to do this is simple as well, and follows the same idea as the god-mode exploit with the only difference being that we want to underflow this time, as we do not want to continue the animation after "BT_ARO_Skill_Cannon_JetAtt_Ing" as this animation is stuck in mid air and travels forward at a high velocity.

Settings the animation to 0 once the player is in said animation will trigger this effect.

**Possible solution for this**
As said previously about the god-mode exploit there needs to be a limiter for animations being executed.

**-----**

**How to interact with a NPC without having the part of the area loaded by the client**

This is game breaking especially due to trading being used to make automatic money by silver selling services.
My method uses the instant teleportation exploit which is also mentioned in this writeup, as well as packet based interactions for NPC's.

The first part of making this work is having positions to teleport to setup as well as the NPC name, once you have this information you can simply teleport to the target and send a packet to open a server-side dialog with the wanted NPC by either the actor-key or by checking nearby NPC's once the client initially changes the position (not visually).

The client doesn't display any interface of anything going on, the server thinks there is a open dialog to said NPC, so the user can send buy item/sell item requests using packets.

**Possible solution for this**
Not much can be done here afaik. except limiting/fixing the instant teleportation exploit.

**-----**

**How to bypass the indicator packet which is sent when a trial-type character exits the battle-arena**

This shouldn't be client sided to begin with, there should be a check in place on the server checking for the clients position leaving the designated area of the battle-arena, leaving this to the client is not very good as this shows.

Where this check sits; is around the area of the selfActor update routine, and can be simply blocked by either patching out this sending part or simply blocking the packet in the sending function for packets.

**Possible solution for this**
Do not have the client checking for this action, let the server decide this purely and either disconnect the user if violated or reset the position forcefully.

**-----**

**How to lock the spawn-position permanently of any trial-type character**

Not exactly sure what goes wrong here but here is my assumption; Once the client enters the savage rift and then exists it using the character selection screen the server will never reset the flag which tells the server to set the spawn position to the battle-arena, causing the client to always spawn at the last position used outside of the battle arena.

How to do it; first of all it is required to escape the battle arena using one of the many methods mentioned, then going to the position wanted to be locked.

Once at the position, the user needs to enter the savage rift - and then swap into the character selection.
The next time the user logs into the trial-type charater it will be stuck at the last position and will never unlock itself.

**Possible solution for this**
Fixing whatever went wrong with this, or checking the logon position for the user, if it is not the battle-arena there is something fishy going on.

**-----**

**How to achieve unlimited underwater breathing**

Very simple as well, here we use the packet which adds breath to us which we can log and re-send.

**Possible solution for this**
Actually check if we are in the water and should be able to gain breath.

**-----**

**How to loot anything within range without any animation**

This should be clear, you can either request the loot list for a dead actor by using packets which will give us results faster, or use the in-game function to do so which does not check for the animation either.

A example of this can be found in the looting.cpp.

**Possible solution for this**
Check for the users current animation on the server - if it does not match the pickup animation there is something fishy going on

**-----**

**How to bypass pet looting cooldowns**

A already well known type of exploit, we simply use the packets which are responsible for pulling the pet in/out and changing the pet's looting speed.

The reason why this works is because it forces the server to reset the internal timer which is responsible to keep track of the last looting event.

Here is a example of how to achieve this;

```cpp
		game::send_packet(seal.data, seal.opcode, seal.size);
		game::send_packet(unseal.data, unseal.opcode, unseal.size);
		game::send_packet(speed.data, speed.opcode, speed.size);
```

This needs to be executed after we recieve a packet of a pet using the looting skill, as well as sending the original packet beforehand (skill use).

**Possible solution for this**
Make sure that the timer does not get reset even when the pet is being reset.

**-----**

**How to bypass the requirement of having full-hp and being idle in order of entering/exiting battle-arena**

Very simple as well just like the other packet based exploits.

We send the packet used to enter/exit BA, and the server will gladly accept it in any state.

**Possible solution for this**
Enforce the limitation server-sided not client-sided.

**-----**

There might be more things I have found over time, or is not worth mentioning.


