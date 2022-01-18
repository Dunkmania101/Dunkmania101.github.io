---
layout: post
title: Saoirse - My sandbox game project
subtitle: The state of my Saoirse game's development.
categories: Saoirse
tags: Saoirse
---

Hello again! This is post 3 on the development progress of my Saoirse (pronounced "Seer-Sha") game.

Here's a quick summary of what I've done since the last post:

    1. I fixed the issue with the invalid check and the infinite recusive function call.

    2. I removed the invalid and blank value system altogether, which is how I achieved the fix in change one.

The removal of the invalid and blank value system should simplify a lot of code and shouldn't be much of a loss.


Here's the current code:

```
# This file is:
# src/python/saoirse_server.py

from dataclasses import dataclass

@dataclass(frozen=True)
class Logger():
    log_level_info="info"
    log_level_warn="warn"
    log_level_error="error"

    def log_msg(sender="Nobody", msg="This is a test message, please ignore it!", level=log_level_info, should_print=True):
        formatted_msg = f"Log: Sender = {sender}: Level = {level}: Message =\n{msg}\n"
        if should_print:
            print(formatted_msg)
        return formatted_msg


class VarHolder():
    def __init__(self):
        pass


class Identifier():
    def __init__(self, path="__blank__", delimiter="/"):
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

    def get_id_from_str_or_id(ide):
        if isinstance(ide, str) or isinstance(ide, list):
            return Identifier(ide)
        return ide


class IdentifierObjGetter():
    def __init__(self, id_in=Identifier(), obj_getter_in=None):
        self.set_id(id_in)

        self.obj_getter = obj_getter_in

    def is_equal(self, other_in):
        return isinstance(other_in, IdentifierObjGetter) and self.get_id().is_equal(other_in.get_id()) and self.get_obj() == other_in.get_obj()

    def copy(self):
        return IdentifierObjGetter(self.get_obj(), self.get_id().copy())

    def set_id(self, id_in):
        self.id = Identifier.get_id_from_str_or_id(id_in)

    def get_id(self):
        return self.id

    def get_obj(self):
        if callable(self.obj_getter):
            return self.obj_getter()
        return None

    def __str__(self):
        return f"{self.id.get_path_str()} : {self.obj_getter}"


class BaseRegistry():
    def __init__(self):
        self.entries = {}

    def get_entries(self):
        return self.entries

    def get_entry(self, id_in):
        ide = Identifier.get_id_from_str_or_id(id_in)

        if self.contains_id(ide):
            return self.entries[ide.get_path_str()]
        return None

    def contains_id(self, id_in):
        ide = Identifier.get_id_from_str_or_id(id_in)
        return ide.get_path_str() in self.get_entries().keys()

    def register_id_obj(self, id_in, obj_getter_in):
        if isinstance(id_in, Identifier):
            self.register_id_obj_pair(IdentifierObjGetter(id_in, obj_getter_in))

    def register_id_obj_pair(self, id_obj_pair_in):
        if isinstance(id_obj_pair_in, IdentifierObjGetter):
            ide = id_obj_pair_in.get_id()
            if isinstance(ide, Identifier):
                id_str = ide.get_path_str()
                if id_str in self.get_entries().keys():
                    Logger.log_msg(sender="Registry", msg=f"Failed to register {id_obj_pair_in} as its id of {id_str} is alread registered!", level=Logger.log_level_warn)
                else:
                    self.entries[id_str] = id_obj_pair_in
        else:
            Logger.log_msg(sender="Registry", msg=f"Failed to register {id_obj_pair_in} as it is not an IdentifierObjectGetter!", level=Logger.log_level_warn)


class BaseCategorizedRegistry(BaseRegistry):
    def __init__(self):
        super().__init__()

    def register_category(self, category_id_in):
        category_id = Identifier.get_id_from_str_or_id(category_id_in)
        self.register_id_obj(category_id, BaseRegistry())

    def get_categories(self):
        return super().get_entries()

    def get_entries(self):
        entries_out = {}
        for key in self.get_categories().keys():
            category = self.get_categories()[key]
            if isinstance(category, BaseRegistry):
                for entry in category.get_entries():
                    if isinstance(entry, IdentifierObjGetter):
                        categorized_entry = entry.copy()
                        entry_id = categorized_entry.get_id().copy()
                        entry_id_path = entry_id.get_path()
                        entry_id_path.insert(0, key)
                        categorized_entry.set_id(Identifier(entry_id_path, entry_id.get_delimiter()))
                        entries_out[categorized_entry.get_id().get_path_str()] = categorized_entry
        return entries_out

    def get_category(self, category_id_in):
        category_id = Identifier.get_id_from_str_or_id(category_id_in)
        if self.contains_id(category_id):
            return self.categories[category_id_in]
        return None

    def register_id_obj_getter_under_category(self, id_obj_pair_in, category_id_in):
        if isinstance(id_obj_pair_in, IdentifierObjGetter):
            if category_id_in not in self.get_categories().keys():
                self.register_category(category_id_in)
            self.register_id_obj_pair(IdentifierObjGetter(id_obj_pair_in, Identifier([category_id_in].extend(id_obj_pair_in.get_id().get_path()))))

    def register_id_obj_under_category(self, id_in, category_id_in, obj_getter_in):
        self.register_id_obj_getter_under_category(IdentifierObjGetter(id_in, obj_getter_in), category_id_in)

    def get_id_obj_pair_under_category(self, category_id_in, obj_id_in):
        category_id = Identifier.get_id_from_str_or_id(category_id_in)
        obj_id = Identifier.get_id_from_str_or_id(obj_id_in)
        return self.get_entry(category_id.extend(obj_id_in, False))


class ThreeDimensionalPosition():
    def __init__(self, x, y, z):
        self.x = x
        self.y = y
        self.z = z

    def get_origin():
        return ThreeDimensionalPosition(0, 0, 0)

    def get_x(self):
        return self.x

    def get_y(self):
        return self.y

    def get_z(self):
        return self.z

    @dataclass(frozen=True)
    class Directions():
        up="up"
        down="down"
        north="north"
        south="south"
        east="east"
        west="west"

    def offset(self, direction, distance):
        if direction == ThreeDimensionalPosition.Directions.up:
            return ThreeDimensionalPosition(self.get_x(), self.get_y() + distance, self.get_z())

        if direction == ThreeDimensionalPosition.Directions.down:
            return ThreeDimensionalPosition(self.get_x(), self.get_y() - distance, self.get_z())

        if direction == ThreeDimensionalPosition.Directions.north:
            return ThreeDimensionalPosition(self.get_x() + distance, self.get_y(), self.get_z())

        if direction == ThreeDimensionalPosition.Directions.south:
            return ThreeDimensionalPosition(self.get_x() - distance, self.get_y(), self.get_z())

        if direction == ThreeDimensionalPosition.Directions.east:
            return ThreeDimensionalPosition(self.get_x(), self.get_y(), self.get_z() + distance)

        if direction == ThreeDimensionalPosition.Directions.west:
            return ThreeDimensionalPosition(self.get_x(), self.get_y(), self.get_z() - distance)

        return self

    def get_dict(self):
        return {"x": self.get_x(), "y": self.get_y(), "z": self.get_z()}

    def get_str(self):
        return str(self.get_dict())

    def from_dict(pos_dict):
        return ThreeDimensionalPosition(pos_dict.get("x"), pos_dict.get("y"), pos_dict.get("z"))

    def from_str(pos_str):
        return ThreeDimensionalPosition.from_dict(dict(pos_str))

    def __str__(self):
        return self.get_str()

    def __eq__(self, other):
        return self.get_x() == other.get_x() and self.get_y() == other.get_y() and self.get_z() == other.get_z()


class GameObject():
    def __init__(self, ide, dimension=None):
        self.ide = ide
        self.dimension = dimension

    def get_id(self):
        return self.ide

    def get_dimension(self):
        return self.dimension

    def tick(self):
        pass


class SpaceGameObject(GameObject):
    def __init__(self, ide, pos=ThreeDimensionalPosition.get_origin(), dimension=None):
        super().__init__(ide, dimension=dimension)
        self.pos = pos

    def set_pos(self, pos):
        self.pos = pos

    def get_pos(self):
        return self.pos

    def has_gravity(self):
        return True


class Tile(SpaceGameObject):
    def __init__(self, ide, pos=ThreeDimensionalPosition.get_origin(), dimension=None):
        super().__init__(ide, pos, dimension)


class Entity(SpaceGameObject):
    def __init__(self, ide, pos=ThreeDimensionalPosition.get_origin(), dimension=None):
        super().__init__(ide, pos, dimension)


class Item(GameObject):
    def __init__(self, ide, dimension=None):
        super().__init__(ide, dimension)


class ThreeDimensionalSpace():
    def __init__(self, dimension):
        self.tiles = []
        self.gravity_speed = 1
        self.dimension = dimension

    def get_tiles(self):
        return self.tiles

    def get_gravity_speed(self):
        return self.gravity_speed

    def get_dimension(self):
        return self.dimension

    def add_tile_at_pos(self, pos, tile):
        tile.set_pos(pos)
        tile.set_world(self.get_dimension())
        self.tiles.append(tile)

    def remove_tile_at_pos(self, pos, check_tile=None):
        for tile in self.get_tiles_at_pos(pos, check_tile):
            self.tiles.remove(tile)

    def set_tile_at_pos(self, pos, tile):
        self.remove_tile_at_pos(pos, tile)
        self.add_tile_at_pos(pos, tile)

    def get_tiles_at_pos(self, pos, check_tile=None):
        tiles = []
        for tile in self.get_tiles():
            if tile.get_pos() == pos and (check_tile == tile or check_tile is None):
                tiles.append(tile)
        return tiles

    def tick_tile_gravity(self, tile):
        if tile.has_gravity():
            tile.set_pos(tile.get_pos().offset(ThreeDimensionalPosition.Directions.down, self.get_gravity_speed()))

    def tick(self):
        for tile in self.get_tiles():
            tile.tick()
            self.tick_tile_gravity(tile)
```


```
# This file is:
# src/python/saoirse_server.py

from saoirse_base import VarHolder, Identifier, IdentifierObjGetter, BaseCategorizedRegistry, Logger, GameObject, Item, Tile, Entity
from dataclasses import dataclass

saoirse_id = "saoirse"

class SaoirseRegistry(BaseCategorizedRegistry):
    def __init__(self):
        super().__init__()

        self.category_tags = self.SaoirseRegistryCategories()

        self.items = VarHolder()
        self.items.hi = ""

        self.register_items()
        self.register_tiles()
        self.register_entities()

    def register_game_obj_under_category(self, ide, category_ide, game_obj_getter):
        if isinstance(game_obj_getter, GameObject):
            self.register_id_obj_under_category(ide, category_ide, game_obj_getter)
        return ide

    def register_items(self):
        self.register_category(self.category_tags.items)

        for ide_str in ["dirt", "stone", "gravel", "oak_log"]:
            ide = Identifier([saoirse_id, ide_str])
            self.register_item(ide, lambda: Item(ide))

    def register_item(self, ide, item_obj_getter=None):
        if item_obj_getter is None:
            item_obj_getter = lambda: Item(ide)
        self.register_id_obj_under_category(ide, self.category_tags.items, item_obj_getter)

    def register_tiles(self):
        self.register_category(self.category_tags.tiles)

    def register_tile(self, ide, tile_obj):
        self.register_id_obj_under_category(ide, self.category_tags.tiles, tile_obj)

    def register_entities(self):
        self.register_category(self.category_tags.entities)

    def register_entity(self, ide, entity_obj):
        self.register_id_obj_under_category(ide, self.category_tags.entities, entity_obj)

    @dataclass(frozen=True)
    class SaoirseRegistryCategories():
        items = "items"
        tiles = "tiles"
        entities = "entities"


registry = SaoirseRegistry()

Logger.log_msg(saoirse_id, registry.get_categories())
Logger.log_msg(saoirse_id, registry.get_entries())

class SaoriseDimension():
    def __init__(self):
        pass
```
