---
layout: post
title: Saoirse - My sandbox game project
subtitle: The state of my Saoirse game's development.
categories: Saoirse
tags: Saoirse
---

Hello again! This is post 4 on the development progress of my Saoirse (pronounced "Seer-Sha") game.

Here's a quick summary of what I've done since the last post:

    1. I added the basis for the client component. It uses the *[pyglet](http://pyglet.org/)* library.

    2. I added some sample entries and a few more categories to the base game's registry.

    3. I switched to Python's built-in logging system, which I didn't know about before.

    4. I fixed a NoneTypeException with the base registry classes that might not have even existed in the last update.


It's been a while since the last post and I might have missed a few details. The summary I've given should at least be mostly complete, though.


Here's the current code:

```
# This file is:
# src/python/saoirse_base.py

import time, os, logging
from enum import Enum

logger = logging.getLogger("saoirse")
#class Logger():
#    class LogLevels(Enum):
#        LOG_LEVEL_INFO="info"
#        LOG_LEVEL_WARN="warn"
#        LOG_LEVEL_ERROR="error"
#
#    def log_msg(sender="Nobody", msg="This is a test message, please ignore it!", level=LogLevels.LOG_LEVEL_INFO, should_print=True):
#        formatted_msg = f"Log: Sender = {sender}: Level = {level}: Message =\n{msg}\n"
#        if should_print:
#            print(formatted_msg)
#        return formatted_msg


#class VarHolder():
#    def __init__(self):
#        pass


class Identifier():
    def __init__(self, path="", delimiter="/"):
        self.set_delimiter(delimiter)
        self.set_path(path)

    def set_path(self, new_path_in, update_self=True):
        new_path = new_path_in
        if isinstance(new_path_in, list):
            for i, part in enumerate(new_path):
                if isinstance(part, str):
                    if self.get_delimiter() in part:
                        fixed = part.split(self.get_delimiter())
                        new_path.pop(i)
                        fixed.reverse()
                        for fixed_part in fixed:
                            new_path.insert(i, fixed_part)
        elif isinstance(new_path_in, str):
            new_path = new_path_in.split(self.get_delimiter())

        if update_self:
            self.path = new_path
            return self
        return Identifier(new_path)

    def set_delimiter(self, new_delimiter_in, update_self=True):
        if isinstance(new_delimiter_in, str):
            new_delimiter = new_delimiter_in
        else:
            new_delimiter = "/"

        if update_self:
            self.delimiter = new_delimiter
            return self
        return Identifier(delimiter=new_delimiter)

    def get_path(self):
        return self.path

    def get_delimiter(self):
        return self.delimiter

    def get_path_str(self):
        return self.get_delimiter().join(self.get_path())

    def is_equal(self, other_in):
        return isinstance(other_in, Identifier) and other_in.get_path() == self.get_path()

    def append(self, other_path_in, update_self=True):
        if isinstance(other_path_in, list):
            new_path = self.path.copy()
            new_path.extend(other_path_in)
            if update_self:
                self.set_path(new_path)
                return self
            return Identifier(new_path)

    def extend(self, other_in, update_self=True):
        if isinstance(other_in, Identifier):
                new_path = self.path.copy()
                new_path.extend(other_in.get_path())
                if update_self:
                    self.set_path(new_path)
                    return self
                return Identifier(new_path)

    def copy(self):
        return Identifier(self.get_path(), self.get_delimiter())

    def get_file_path(self):
        return os.path.join(*self.get_path())

    def get_id_from_str_list_or_id(ide):
        if isinstance(ide, str) or isinstance(ide, list):
            return Identifier(ide)
        elif isinstance(ide, Identifier):
            return ide
        return None


class IdentifierObjPair():
    def __init__(self, id_in=Identifier(), obj_in=None):
        self.set_id(id_in)

        self.obj = obj_in

    def is_equal(self, other_in):
        return isinstance(other_in, IdentifierObjPair) and self.get_id().is_equal(other_in.get_id()) and self.get_obj() == other_in.get_obj()

    def copy(self):
        return IdentifierObjPair(self.get_obj(), self.get_id().copy())

    def set_id(self, id_in):
        self.id = Identifier.get_id_from_str_list_or_id(id_in)

    def get_id(self):
        return self.id

    def get_obj(self):
        return self.obj

    def __str__(self):
        return f"{self.id.get_path_str()} : {self.obj}"


class BaseRegistry():
    def __init__(self):
        self.entries = {}

    def get_entries_dict(self):
        return self.entries

    def get_entries(self):
        return self.entries.values()

    def get_entry(self, id_in):
        ide = Identifier.get_id_from_str_list_or_id(id_in)

        if self.contains_id(ide):
            return self.entries[ide.get_path_str()]
        return None

    def contains_id(self, id_in):
        ide = Identifier.get_id_from_str_list_or_id(id_in)
        return ide.get_path_str() in self.get_entries_dict().keys()

    def register_id_obj(self, id_in, obj_in):
        if isinstance(id_in, Identifier):
            self.register_id_obj_pair(IdentifierObjPair(id_in, obj_in))

    def register_id_obj_pair(self, id_obj_pair_in):
        if isinstance(id_obj_pair_in, IdentifierObjPair):
            ide = id_obj_pair_in.get_id()
            if isinstance(ide, Identifier):
                id_str = ide.get_path_str()
                if id_str in self.get_entries_dict().keys():
                    logger.warning(msg=f"Failed to register {id_obj_pair_in} as its id of {id_str} is alread registered!")
                else:
                    self.entries[id_str] = id_obj_pair_in
        else:
            logger.warning(msg=f"Failed to register {id_obj_pair_in} as it is not an IdentifierObjectPair!")


class BaseCategorizedRegistry(BaseRegistry):
    def __init__(self):
        super().__init__()

    def register_category(self, category_id_in):
        category_id = Identifier.get_id_from_str_list_or_id(category_id_in)
        super().register_id_obj(category_id, BaseRegistry())

    #def get_entries(self):
    #    entries_out = {}
    #    for key in self.entries.keys():
    #        category = self.entries[key]
    #        if isinstance(category, BaseRegistry):
    #            for entry in category.get_entries():
    #                if isinstance(entry, IdentifierObjPair):
    #                    categorized_entry = entry.copy()
    #                    entry_id = categorized_entry.get_id().copy()
    #                    entry_id_path = entry_id.get_path()
    #                    entry_id_path.insert(0, key)
    #                    categorized_entry.set_id(Identifier(entry_id_path, entry_id.get_delimiter()))
    #                    entries_out[categorized_entry.get_id().get_path_str()] = categorized_entry
    #    return entries_out

    def get_category(self, category_id_in):
        return super().get_entry(category_id_in).get_obj()

    def register_id_obj_pair_under_category(self, id_obj_pair_in, category_id_in):
        if isinstance(id_obj_pair_in, IdentifierObjPair):
            if category_id_in.get_path_str() not in self.get_entries_dict().keys():
                self.register_category(category_id_in)
            self.get_category(category_id_in).register_id_obj_pair(id_obj_pair_in)

    def register_id_obj_under_category(self, id_in, category_id_in, obj_pair_in):
        self.register_id_obj_pair_under_category(IdentifierObjPair(id_in, obj_pair_in), category_id_in)

    def get_id_obj_pair_under_category(self, category_id_in, obj_id_in):
        category_id = Identifier.get_id_from_str_list_or_id(category_id_in)
        obj_id = Identifier.get_id_from_str_list_or_id(obj_id_in)
        return self.get_entry(category_id).get_obj().get_entry(obj_id)


class ThreeDimensionalPosition():
    def __init__(self, x=0, y=0, z=0):
        self.x = x
        self.y = y
        self.z = z

    def get_origin():
        return ThreeDimensionalPosition()

    def get_x(self):
        return self.x

    def get_y(self):
        return self.y

    def get_z(self):
        return self.z

    def get_distance_from_other(self, other):
        return ((self.get_x() + other.get_x())**2 + (self.get_y() + other.get_y())**2 + (self.get_z() + other.get_z())**2)**0.5

    def get_distance_two_1d_points(primary, secondary):
        return secondary - primary

    def get_distance_from_other_x(self, other):
        return ThreeDimensionalPosition.get_distance_two_1d_points(self.get_x(), other.get_x())

    def get_distance_from_other_y(self, other):
        return ThreeDimensionalPosition.get_distance_two_1d_points(self.get_y(), other.get_y())

    def get_distance_from_other_z(self, other):
        return ThreeDimensionalPosition.get_distance_two_1d_points(self.get_z(), other.get_z())

    class Direction(Enum):
        def of_index(i=1):
            if i < -6 or i > 6:
                i %= 6
            if i == 0:
                i = 6
            return getattr(ThreeDimensionalPosition.Direction, abs(i))

        def get_opposite(self):
            if self == ThreeDimensionalPosition.Direction.UP:
                return ThreeDimensionalPosition.Direction.DOWN
            elif self == ThreeDimensionalPosition.Direction.DOWN:
                return ThreeDimensionalPosition.Direction.UP
            elif self == ThreeDimensionalPosition.Direction.FRONT:
                return ThreeDimensionalPosition.Direction.BACK
            elif self == ThreeDimensionalPosition.Direction.BACK:
                return ThreeDimensionalPosition.Direction.FRONT
            elif self == ThreeDimensionalPosition.Direction.LEFT:
                return ThreeDimensionalPosition.Direction.RIGHT
            return ThreeDimensionalPosition.Direction.LEFT

        def get_relative(self, front, up):
            if front == up or front.get_opposite() == up:
                return None
            dist_front = ThreeDimensionalPosition.Direction.FRONT.value - self.value
            dist_up = ThreeDimensionalPosition.Direction.UP.value - self.value
            for d in ThreeDimensionalPosition.Direction:
                if front.value - d.value == dist_front and up.value - d.value == dist_up:
                    return d
            return None

        FRONT = 1
        LEFT = 2
        BACK = 3
        UP = 4
        RIGHT = 5
        DOWN = 6

    def offset(self, direction, distance):
        if direction == ThreeDimensionalPosition.Direction.UP:
            return ThreeDimensionalPosition(self.get_x(), self.get_y() + distance, self.get_z())

        if direction == ThreeDimensionalPosition.Direction.DOWN:
            return ThreeDimensionalPosition(self.get_x(), self.get_y() - distance, self.get_z())

        if direction == ThreeDimensionalPosition.Direction.FRONT:
            return ThreeDimensionalPosition(self.get_x() + distance, self.get_y(), self.get_z())

        if direction == ThreeDimensionalPosition.Direction.BACK:
            return ThreeDimensionalPosition(self.get_x() - distance, self.get_y(), self.get_z())

        if direction == ThreeDimensionalPosition.Direction.RIGHT:
            return ThreeDimensionalPosition(self.get_x(), self.get_y(), self.get_z() + distance)

        if direction == ThreeDimensionalPosition.Direction.LEFT:
            return ThreeDimensionalPosition(self.get_x(), self.get_y(), self.get_z() - distance)

        return self

    def get_nearest_direction_to_other_pos(self, other):
        dist_x = self.get_distance_from_other_x(other)
        dist_y = self.get_distance_from_other_y(other)
        dist_z = self.get_distance_from_other_z(other)

        dist_farthest = max(abs(dist_x), abs(dist_y), abs(dist_z))
        if dist_farthest == abs(dist_x):
            if dist_x < 0:
                return ThreeDimensionalPosition.Direction.LEFT
            return ThreeDimensionalPosition.Direction.RIGHT
        elif dist_farthest == abs(dist_y):
            if dist_y < 0:
                return ThreeDimensionalPosition.Direction.DOWN
            return ThreeDimensionalPosition.Direction.UP
        else:
            if dist_z < 0:
                return ThreeDimensionalPosition.Direction.BACK
            return ThreeDimensionalPosition.Direction.FRONT

    class Axies(Enum):
        X = "x"
        Y = "y"
        Z = "z"

    def to_dict(self):
        return {ThreeDimensionalPosition.Axies.X: self.get_x(), ThreeDimensionalPosition.Axies.Y: self.get_y(), ThreeDimensionalPosition.Axies.Z: self.get_z()}

    def to_str(self):
        return str(self.to_dict())

    def of_dict(pos_dict):
        return ThreeDimensionalPosition(pos_dict.get(ThreeDimensionalPosition.Axies.X, 0), pos_dict.get(ThreeDimensionalPosition.Axies.Y, 0), pos_dict.get(ThreeDimensionalPosition.Axies.Z, 0))

    def of_str(pos_str):
        return ThreeDimensionalPosition.of_dict(dict(pos_str))

    def __str__(self):
        return self.to_str()

    def __eq__(self, other):
        return self.get_x() == other.get_x() and self.get_y() == other.get_y() and self.get_z() == other.get_z()


class ActionIds(Enum):
    MAIN = "main"
    SECONDARY = "secondary"


class TickableObject():
    def tick(self):
        pass

    def get_ticks_per_second(self):
        return 64


class MainGameObject(TickableObject):
    def __init__(self, ide, server):
        self.ide = ide
        self.server = server
        self.on_created()

    def set_removed(self, removed=True):
        self.removed = removed

    def on_created(self):
        self.set_removed(False)

    def on_removed(self):
        self.set_removed(True)

    def get_server(self):
        return self.server

    def get_id(self):
        return self.ide

    def set_data(self, data):
        pass

    def get_data(self):
        return {}


class InteractableGameObject():
    def get_action_by_id(self, ide, actor):
        return None

    def get_main_action(self, actor):
        return self.get_action_by_id(ActionIds.MAIN, actor)

    def get_secondary_action(self, actor):
        return self.get_action_by_id(ActionIds.SECONDARY, actor)


class SpaceGameObject(MainGameObject, InteractableGameObject):
    def __init__(self, ide, server, pos=ThreeDimensionalPosition.get_origin(), three_dimensional_space=None):
        super().__init__(ide, server)

        self.set_pos(pos)
        self.set_three_dimensional_space(three_dimensional_space)

    def set_pos(self, pos):
        self.pos = pos

    def get_pos(self):
        return self.pos

    def set_three_dimensional_space(self, three_dimensional_space):
        self.three_dimensional_space = three_dimensional_space

    def get_three_dimensional_space(self):
        return self.three_dimensional_space

    def has_gravity(self):
        return True

    def get_mass(self):
        return 1


class Item(SpaceGameObject):
    def __init__(self, ide, server, pos=ThreeDimensionalPosition.get_origin(), three_dimensional_space=None):
        super().__init__(ide, server, pos, three_dimensional_space)


class Tile(SpaceGameObject):
    def __init__(self, ide, server, pos=ThreeDimensionalPosition.get_origin(), three_dimensional_space=None):
        super().__init__(ide, server, pos, three_dimensional_space)


class Fluid(SpaceGameObject):
    def __init__(self, ide, server, pos=ThreeDimensionalPosition.get_origin(), three_dimensional_space=None):
        super().__init__(ide, server, pos, three_dimensional_space)


class Entity(SpaceGameObject):
    def __init__(self, ide, server, pos=ThreeDimensionalPosition.get_origin(), three_dimensional_space=None):
        super().__init__(ide, server, pos, three_dimensional_space)

    def get_agenda(self):
        return []


class ThreeDimensionalSpace(SpaceGameObject):
    def __init__(self, ide, server, three_dimensional_space=None):
        super().__init__(ide, server, three_dimensional_space)

        self.space_game_objects = {}

    def get_server(self):
        return self.server

    def get_space_game_objects_dict(self):
        return self.space_game_objects

    def get_space_game_objects(self):
        return self.get_space_game_objects().values()

    def get_g_constant(self):
        return 6.67 * (10**-11)

    def get_gravity_speed(self, m1, m2, distance):
        return ((self.get_g_constant() * m1 * m2) / (distance**2)) / self.get_server().get_ticks_per_second()

    def add_space_game_object_at_pos(self, pos, space_game_object):
        space_game_object.set_pos(pos)
        space_game_object.set_three_dimensional_space(self)
        pos_str = pos.get_str()
        existing_objects = self.space_game_objects.get(pos_str, [])
        existing_objects.append(space_game_object)
        self.space_game_objects[pos_str] = existing_objects

    def remove_space_game_object_at_pos(self, pos, check_space_game_object=None):
        for space_game_object in self.get_space_game_objects_at_pos(pos, check_space_game_object):
            self.space_game_objects.pop(space_game_object)

    def replace_space_game_object_at_pos(self, pos, old_space_game_object, new_space_game_object):
        self.remove_space_game_object_at_pos(pos, old_space_game_object)
        self.add_space_game_object_at_pos(pos, new_space_game_object)

    def get_space_game_objects_at_pos(self, pos, check_space_game_object=None):
        if check_space_game_object is None:
            return self.get_space_game_objects_dict().get(pos.get_str(), [])
        space_game_objects = []
        for space_game_object in self.get_space_game_objects():
            if space_game_object.get_pos() == pos and (check_space_game_object == space_game_object or check_space_game_object is None):
                space_game_objects.append(space_game_object)
        return space_game_objects

    def get_nearest_obj_to_pos(self, pos, exclusions=[]):
        if len(self.get_space_game_objects()) > 0:
            nearest = self.get_space_game_objects()[0]
            if len(self.get_space_game_objects()) > 1:
                for obj in self.get_space_game_objects()[1:]:
                    if obj not in exclusions and obj.get_pos().get_distance_from_other(pos) < nearest.get_pos().get_distance_from_other(pos):
                        nearest = obj
            if nearest not in exclusions:
                return nearest
        return None

    def tick_space_game_object_gravity(self, space_game_object):
        if len(self.get_space_game_objects()) > 1 and space_game_object.has_gravity():
            nearest = self.get_nearest_obj_to_pos(space_game_object.get_pos(), [space_game_object])
            space_game_object.set_pos(space_game_object.get_pos().offset(space_game_object.get_pos().get_nearest_direction_to_other_pos(nearest.get_pos()), self.get_gravity_speed(space_game_object.get_mass(), nearest.get_mass(), nearest.get_pos().get_distance_from_other(space_game_object.get_pos()))))

    def tick(self):
        for space_game_object in self.get_space_game_objects():
            space_game_object.tick()
            self.tick_space_game_object_gravity(space_game_object)

    class SaveDataKeys(Enum):
        IDE = "ide"
        POS = "pos"
        DATA = "data"

    def set_data(self, data):
        for space_game_object_data in data.values():
            if ThreeDimensionalSpace.SaveDataKeys.IDE in space_game_object_data.keys() and ThreeDimensionalSpace.SaveDataKeys.POS in space_game_object_data.keys():
                ide = Identifier(space_game_object_data[ThreeDimensionalSpace.SaveDataKeys.IDE])
                space_game_object_pair = self.get_server().get_registry().get_entry(ide)
                if space_game_object_pair is not None:
                    pos = ThreeDimensionalPosition.of_dict(space_game_object_data[ThreeDimensionalSpace.SaveDataKeys.POS])
                    space_game_object = space_game_object_pair.get_obj()(ide, pos, self)
                    if space_game_object is not None:
                        space_game_object.set_data(space_game_object_data.get(ThreeDimensionalSpace.SaveDataKeys.DATA), {})
                        self.add_space_game_object_at_pos(pos, space_game_object)

    def get_data(self):
        data = {}
        for i, space_game_object in enumerate(self.get_space_game_objects()):
            if space_game_object is not None:
                ide_path = space_game_object.get_id().get_path()
                pos_dict = space_game_object.get_pos().get_dict()
                saved_data = space_game_object.get_data()
                space_game_object_data = {}
                space_game_object_data[ThreeDimensionalSpace.SaveDataKeys.IDE] = ide_path
                space_game_object_data[ThreeDimensionalSpace.SaveDataKeys.POS] = pos_dict
                space_game_object_data[ThreeDimensionalSpace.SaveDataKeys.DATA] = saved_data
                data[str(i)] = space_game_object_data
        return data


class BaseServer(MainGameObject):
    def __init__(self, ide, registry):
        super().__init__(ide, self)

        self.registry = registry
        self.registry.server = self
        self.spaces = {}

    def get_registry(self):
        return self.registry

    def get_spaces(self):
        return self.spaces

    def add_three_dimensional_space(self, three_dimensional_space):
        self.spaces[three_dimensional_space.get_id().get_str()] = three_dimensional_space

    def get_three_dimensional_space(self, ide):
        return self.spaces.get(ide.get_str(), None)

    def tick(self):
        for space in self.get_spaces():
            space.tick()
        time.sleep(1 / self.get_ticks_per_second())

    def get_data(self):
        data = {}
        for space in self.get_spaces():
            data[space.get_id().get_str()] = space.get_data()
        return data
```

```
# This file is:
# src/python/saoirse_server.py

import sys
from enum import Enum
from saoirse_base import Identifier, BaseCategorizedRegistry, MainGameObject, Item, Tile, Fluid, Entity, BaseServer

saoirse_id = "saoirse"

class SaoirseRegistry(BaseCategorizedRegistry):
    def __init__(self, server):
        super().__init__()

        self.server = server

        self.register_items()
        self.register_tiles()
        self.register_fluids()
        self.register_entities()

    def get_server(self):
        return self.server

    def register_game_obj_under_category(self, ide, category_ide, game_obj_pair):
        if isinstance(game_obj_pair, MainGameObject):
            self.register_id_obj_under_category(ide, category_ide, game_obj_pair)
        return ide

    def register_items(self):
        self.register_category(SaoirseRegistry.SaoirseRegistryCategories.ITEMS.value)

        for ide_str in [
                "pebble",
                "paper",
                ##########
                "silkworm",
                ##########
                "cotton_seed",
                ##########
                "silk_thread",
                "cotton_thread",
                "woolen_thread",
                "canvas_thread",
                #####
                "silk_sheet",
                "cotton_sheet",
                "woolen_sheet",
                "canvas_sheet",
                ##########
                "glass_shard",
                ##########
                "oak_stick",
                "spruce_stick",
                "ironwood_stick",
                "hickory_stick",
                ]:
            ide = Identifier([saoirse_id, ide_str])
            self.register_item(ide, True)                                   

    def register_item(self, ide, item_obj=None):
        if item_obj is None:
            item_obj = lambda: Item(ide, self.get_server())
        self.register_id_obj_under_category(ide, SaoirseRegistry.SaoirseRegistryCategories.ITEMS.value, item_obj)

    def register_tiles(self):
        self.register_category(SaoirseRegistry.SaoirseRegistryCategories.TILES.value)

        for ide_str in [
                "dirt", 
                "gravel",
                "sand",
                "peat",
                ##########
                "granite",
                "gabbro",
                "bassalt",
                "obsidian",
                "limestone",
                "bedrock",
                ##########
                "oak_log",
                "spruce_log",
                "ironwood_log",
                "hickory_log",
                ]:
            ide = Identifier([saoirse_id, ide_str])
            self.register_tile(ide, True)                                   

    def register_tile(self, ide, tile_obj=None, add_item=False, item_obj=None):
        if tile_obj is None:
            tile_obj = lambda: Tile(ide, self.get_server())
        if add_item:
            if item_obj is None:
                item_obj = lambda: Item(ide, self.get_server())
            self.register_item(ide, item_obj)
        self.register_id_obj_under_category(ide, SaoirseRegistry.SaoirseRegistryCategories.TILES.value, tile_obj)

    def register_fluids(self):
        self.register_category(SaoirseRegistry.SaoirseRegistryCategories.FLUIDS.value)

        for ide_str in [
                "water",
                "lava",
                "milk",
                "honey",
                "slime",
                "oil",
                "tar",
        ]:
            ide = Identifier([saoirse_id, ide_str])
            self.register_fluid(ide)

    def register_fluid(self, ide, fluid_obj=None):
        if fluid_obj is None:
            fluid_obj = lambda: Fluid(ide, self.get_server())
        self.register_id_obj_under_category(ide, SaoirseRegistry.SaoirseRegistryCategories.FLUIDS.value, fluid_obj)

    def register_entities(self):
        self.register_category(SaoirseRegistry.SaoirseRegistryCategories.ENTITIES.value)

    def register_entity(self, ide, entity_obj):
        if entity_obj is None:
            entity_obj = lambda: Entity(ide, self.get_server())
        self.register_id_obj_under_category(ide, SaoirseRegistry.SaoirseRegistryCategories.ENTITIES.value, entity_obj)

    class SaoirseRegistryCategories(Enum):
        ITEMS = Identifier([saoirse_id, "items"])
        TILES = Identifier([saoirse_id, "tiles"])
        FLUIDS = Identifier([saoirse_id, "fluids"])
        ENTITIES = Identifier([saoirse_id, "entities"])


class SaoirseServer(BaseServer):
    def __init__(self):
        super().__init__(saoirse_id, SaoirseRegistry(self))


def main(args):
    server = SaoirseServer()
    while True:
        server.tick()


if __name__ == "__main__":
    main(sys.argv)
```

```
# This file is:
# src/python/saoirse_client.py

import time, sys, pyglet
from saoirse_base import Identifier, MainGameObject, InteractableGameObject
from saoirse_server import saoirse_id


class SaoirseClientScreenWidget(MainGameObject, InteractableGameObject):
    def __init__(self, ide, server, parent=None, width=20, height=20, left=0, top=0):
        super().__init__(ide, server)

        self.set_parent(parent)
        self.set_width(width)
        self.set_height(height)
        self.set_left(left)
        self.set_top(top)

    def set_parent(self, parent):
        self.parent = parent

    def get_parent(self):
        return self.parent

    def set_width(self, width):
        self.width = width

    def set_height(self, height):
        self.height = height

    def get_width(self):
        return self.width

    def get_height(self):
        return self.height

    def set_left(self, left):
        self.left = left

    def set_top(self, top):
        self.top = top

    def get_left(self):
        return self.left

    def get_top(self):
        return self.top

    def get_right(self):
        return self.get_left() + self.get_width()

    def get_bottom(self):
        return self.get_top() + self.get_height()

    def draw_image(self, ide, left=0, top=0):
        self.get_parent().draw_image(ide, self.get_top() + top, self.get_left() + left)

    def draw_text(self, text, size, left=0, top=0):
        self.get_parent().draw_text(size, text, self.get_left() + left, self.get_top() + top)

    def play_sound(self, ide):
        self.get_parent().play_sound(ide)

    def draw(self):
        pass

    def tick_content(self):
        pass

    def tick(self):
        self.tick_content()


class SaoirseClientButton(SaoirseClientScreenWidget):
    def __init__(self, ide, server, parent=None, width=20, height=20, left=0, top=0, label=""):
        super().__init__(ide, server, parent=parent, width=width, height=height, left=left, top=top)

        self.label = label

    def get_label(self):
        return self.label

    def tick_content(self):
        self.draw_text(self.get_label(), self.get_width()/2, self.get_left() + self.get_width()/4, self.get_top() + self.get_height()/4)


class SaoirseScreen(SaoirseClientScreenWidget):
    def __init__(self, ide, server, parent=None, width=1200, height=800, left=0, top=0, title=""):
        super().__init__(ide, server, parent, width, height, left, top)

        self.title = title
        self.children = {}

    def get_title(self):
        return self.title

    def get_children(self):
        return self.children

    def add_child(self, child):
        child.set_parent(self)
        child.set_left(child.get_left() + self.get_left())
        child.set_top(child.get_top() + self.get_top())
        self.children[child.get_id().get_path_str()] = child

    def on_removed(self):
        for child in self.get_children().values():
            child.on_removed()
        super().on_removed()

    def draw(self):
        for child in self.get_children().values():
            child.draw()

    def tick(self):
        super().tick()
        for child in self.get_children().values():
            child.tick()


class SaoirseClientWorldScreen(SaoirseScreen):
    def __init__(self, server, parent):
        super().__init__(Identifier([saoirse_id, "screens", "world"]), server, parent, title="")


class SaoirseClientSingleplayerButton(SaoirseClientButton):
    def __init__(self, server, parent=None, width=50, height=20, left=50, top=50):
        super().__init__(Identifier([saoirse_id, "screens", "singleplayer"]), server, parent=parent, width=width, height=height, left=left, top=top, label="Singleplayer")

    def tick_content(self):
        self.draw_image(Identifier(["resources", saoirse_id, "media", "pic1.png"]), -self.left, -self.top)
        super().tick_content()


class SaoirseClientMainScreen(SaoirseScreen):
    def __init__(self, server, parent=None, width=1200, height=800, left=0, top=0, title=""):
        super().__init__(Identifier([saoirse_id, "screens", "main"]), server, parent=parent, width=width, height=height, left=left, top=top, title=title)

    def draw(self):
        self.add_child(SaoirseClientSingleplayerButton(self.get_server()))
        super().draw()


class SaoirseClientMainWindowScreen(SaoirseScreen):
    def __init__(self):
        super().__init__(Identifier([saoirse_id, "screens", "main_window"]), None, self, title="Saoirse")

        self.draw()

        self.window = pyglet.window.Window(width=self.get_width(), height=self.get_height())

        @self.window.event
        def on_draw():
            self.window.clear()
            self.tick()

        pyglet.app.run()

        self.on_removed()

    def draw(self):
        self.add_child(SaoirseClientMainScreen(self.get_server()))
        super().draw()

    def tick(self):
        super().tick()
        time.sleep(1 / self.get_ticks_per_second())

    def tick_content(self):
        pass
        #self.draw_image(Identifier(["resources", saoirse_id, "media", "pic1.png"]), 0, 0)

    def draw_image(self, ide, left, top):
        pyglet.resource.image(ide.get_file_path()).blit(left, top)

    def draw_text(self, text, size, left, top):
        pyglet.text.Label(text,
                          font_name='Exo 2 Regular',
                          font_size=size,
                          x=left, y=top,
                          anchor_x='left', anchor_y='top').draw()

    def play_sound(self, ide):
        pyglet.resource.media(ide.get_file_path()).play()


def main(args):
    SaoirseClientMainWindowScreen()


if __name__ == "__main__":
    main(sys.argv)
```
