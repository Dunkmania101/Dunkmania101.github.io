---
layout: post
title: Saoirse - My sandbox game project
subtitle: The state of my Saoirse game's development.
categories: Saoirse
tags: Saoirse
---

Hello again! As I said in my first post on this blog, I'm making a game called Saoirse.

It's a sandbox game, meaning players are free to do whatever they want within the loose constraints of balance in the game. This gives them a lot of power and freedom, as is the case in existing games like Minecraft or Terraria.


Currently, it's just a bunch of early python classes defining how some basic ideas might end up working, but it's a start and should be fairly easy to translate to Nim, the language I intend to make the final product in.


Here are some snippets:

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


class BlankInvalidOrValue():
    def __init__(self, value_in="__blank__", check_valid_in=True):
        check_valid = not (isinstance(value_in, str) and value_in == BlankInvalidOrValue.invalid_str()) and check_valid_in
        self.set_value(value_in, check_valid)

    def __eq__(self, other_in):
        return self.is_equal(other_in)

    def is_equal(self, other_in):
        return isinstance(other_in, type(self)) and other_in.get_value() == self.get_value()

    def copy(self):
        return BlankInvalidOrValue(self.get_value(), False)

    def set_value_invalid(self):
        self.value = BlankInvalidOrValue.invalid().get_value()

    def get_value(self):
        return self.value

    def is_value_acceptable(self, value_in):
        return True

    def is_value_valid(self, value_in):
        if isinstance(value_in, str) and value_in == BlankInvalidOrValue.invalid_str():
            return False
        return self.is_value_acceptable(value_in)

    def set_value(self, new_value_in, check_valid=True):
        if BlankInvalidOrValue.is_value_valid(self, new_value_in) or not check_valid:
            self.value = new_value_in
        else:
            self.set_value_invalid()

    def blank_str():
        return "__blank__"

    def invalid_str():
        return "__invalid__"

    def blank():
        return BlankInvalidOrValue(BlankInvalidOrValue.blank_str())

    def invalid():
        return BlankInvalidOrValue(BlankInvalidOrValue.invalid_str())

    def is_blank(self):
        return self.is_equal(BlankInvalidOrValue.blank())

    def is_invalid(self):
        return self.is_equal(BlankInvalidOrValue.invalid())


class Identifier(BlankInvalidOrValue):
    def __init__(self, path_in="__blank__", delimiter_in="/"):
        super().__init__(path_in)
        self.set_delimiter(delimiter_in)
        self.set_path(path_in)

    def is_value_valid(value_in):
        return isinstance(value_in, list) and all(isinstance(part, str) and part not in Identifier.invalid().get_path() for part in value_in)

    def set_path_invalid(self):
        self.value = Identifier.invalid().get_path()

    def validate_path(path_in):
        return isinstance(path_in, list) and all(isinstance(part, str) and part not in Identifier.invalid().get_path() for part in path_in)

    def validate_own_path(self, set_if_invalid=True):
        if Identifier.validate_path(self.get_path()):
            return True
        if set_if_invalid:
            self.set_path_invalid()
        return False

    def set_path(self, new_path_in, update_self=True):
        if isinstance(new_path_in, list):
            new_path = new_path_in
            if Identifier.validate_path(new_path):
                for i, part in enumerate(new_path):
                    if isinstance(part, str):
                        if self.get_delimiter() in part:
                            fixed = part.split(self.get_delimiter())
                            new_path.pop(i)
                            fixed.reverse()
                            for fixed_part in fixed:
                                new_path.insert(i, fixed_part)
                    else:
                        new_path = Identifier.invalid().get_path()
            else:
                new_path = Identifier.invalid().get_path()
        elif isinstance(new_path_in, str):
            new_path = new_path_in.split(self.get_delimiter())
        else:
            new_path = Identifier.invalid().get_path()

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
        return Identifier.blank().set_delimiter(new_delimiter, True)

    def get_path(self):
        return self.path

    def get_delimiter(self):
        return self.delimiter

    def get_path_str(self):
        if self.validate_own_path():
            return self.get_delimiter().join(self.get_path())
        return self.get_delimiter().join(Identifier.invalid_str())

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
            if not self.is_invalid() and not other_in.is_invalid():
                new_path = self.path.copy()
                new_path.extend(other_in.get_path())
                if update_self:
                    self.set_path(new_path)
                    return self
                return Identifier(new_path)
        if update_self:
            self.set_path_invalid()
            return self
        return Identifier.invalid()

    def copy(self):
        return Identifier(self.get_path(), self.get_delimiter())

    def blank():
        return Identifier(Identifier.blank_str())

    def invalid():
        return Identifier(Identifier.invalid_str())

    def is_blank(self):
        return self.is_equal(Identifier.blank())

    def is_invalid(self):
        return self.is_equal(Identifier.invalid())

    def get_id_from_str_or_id(ide):
        if isinstance(ide, str):
            return Identifier(ide)
        elif isinstance(ide, Identifier):
            return ide
        else:
            return Identifier.invalid()


class IdentifierObjPair(BlankInvalidOrValue):
    def __init__(self, id_in=Identifier.blank(), obj_in=None):
        self.set_id(id_in)

        super().__init__(self.id)

        self.obj = obj_in

    def is_equal(self, other_in):
        return isinstance(other_in, IdentifierObjPair) and self.get_id().is_equal(other_in.get_id()) and self.get_obj() == other_in.get_obj()

    def copy(self):
        return IdentifierObjPair(self.get_obj(), self.get_id().copy())

    def set_id(self, id_in):
        self.id = Identifier.get_id_from_str_or_id(id_in)

    def get_id(self):
        if isinstance(self.id, Identifier):
            return self.id
        return Identifier.invalid()

    def get_obj(self):
        return self.obj

    def blank():
        return IdentifierObjPair(None, Identifier.blank())

    def invalid():
        return IdentifierObjPair(None, Identifier.invalid())

    def __str__(self):
        return f"{self.id.get_path_str()} : {self.obj}"


class BaseRegistry():
    def __init__(self):
        self.entries = {}

    def get_entries(self):
        return self.entries

    def get_entry(self, id_in):
        ide = Identifier.get_id_from_str_or_id(id_in)
        if ide.is_invalid():
            return IdentifierObjPair.invalid()

        if self.contains_id(ide):
            return self.entries[ide.get_path_str()]
        return IdentifierObjPair.blank()

    def contains_id(self, id_in):
        ide = Identifier.get_id_from_str_or_id(id_in)
        if ide.is_invalid():
            return False
        return ide.get_path_str() in self.get_entries().keys()

    def register_id_obj(self, id_in, obj_in):
        if isinstance(id_in, Identifier):
            self.register_id_obj_pair(IdentifierObjPair(id_in, obj_in))

    def register_id_obj_pair(self, id_obj_pair_in):
        if isinstance(id_obj_pair_in, IdentifierObjPair):
            ide_invalid = True
            ide = id_obj_pair_in.get_id()
            if isinstance(ide, Identifier):
                if not ide.is_invalid():
                    ide_invalid = False
                    id_str = ide.get_path_str()
                    if id_str in self.get_entries().keys():
                        Logger.log_msg(sender="Registry", msg=f"Failed to register {id_obj_pair_in} as its id of {id_str} is alread registered!", level=Logger.log_level_warn)
                    else:
                        self.entries[id_str] = id_obj_pair_in
            if ide_invalid:
                Logger.log_msg(sender="Registry", msg=f"Failed to register {id_obj_pair_in} as its id of {ide} is invalid!", level=Logger.log_level_warn)
        else:
            Logger.log_msg(sender="Registry", msg=f"Failed to register {id_obj_pair_in} as it is invalid!", level=Logger.log_level_warn)


class BaseCategorizedRegistry(BaseRegistry):
    def __init__(self):
        super().__init__()

    def register_category(self, category_id_in):
        category_id = Identifier.get_id_from_str_or_id(category_id_in)
        if not category_id.is_invalid():
            self.register_id_obj(category_id, BaseRegistry())

    def get_categories(self):
        return super().get_entries()

    def get_entries(self):
        entries_out = {}
        for key in self.get_categories().keys():
            category = self.get_categories()[key]
            if isinstance(category, BaseRegistry):
                for entry in category.get_entries():
                    if isinstance(entry, IdentifierObjPair):
                        categorized_entry = entry.copy()
                        entry_id = categorized_entry.get_id().copy()
                        entry_id_path = entry_id.get_path()
                        entry_id_path.insert(0, key)
                        categorized_entry.set_id(Identifier(entry_id_path, entry_id.get_delimiter()))
                        entries_out[categorized_entry.get_id().get_path_str()] = categorized_entry
        return entries_out

    def get_category(self, category_id_in):
        category_id = Identifier.get_id_from_str_or_id(category_id_in)
        if not category_id.is_invalid():
            if self.contains_id(category_id):
                return self.categories[category_id_in]
        return Identifier.invalid()

    def register_id_obj_under_category(self, id_in, category_id_in, obj_in):
        self.register_id_obj_pair_under_cateogry(IdentifierObjPair(id_in, obj_in), category_id_in)

    def register_id_obj_pair_under_category(self, id_obj_pair_in, category_id_in):
        if isinstance(id_obj_pair_in, IdentifierObjPair):
            if category_id_in not in self.get_categories().keys():
                self.register_category(category_id_in)
            self.register_id_obj_pair(IdentifierObjPair(id_obj_pair_in, Identifier(category_id_in)))

    def get_id_obj_pair_under_category(self, category_id_in, obj_id_in):
        category_id = Identifier.get_id_from_str_or_id(category_id_in)
        if category_id.is_invalid():
            return IdentifierObjPair.invalid()

        obj_id = Identifier.get_id_from_str_or_id(obj_id_in)
        if obj_id.is_invalid():
            return IdentifierObjPair.invalid()

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


class Tile(BlankInvalidOrValue):
    def __init__(self, ide, pos=ThreeDimensionalPosition.get_origin(), dimension = None):
        self.ide = ide
        self.pos = pos
        self.dimension = dimension

    def set_pos(self, pos):
        self.pos = pos

    def get_pos(self):
        return self.pos

    def get_id(self):
        return self.ide

    def get_dimension(self):
        return self.dimension

    def has_gravity(self):
        return True

    def tick(self):
        pass


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

from saoirse_base import BlankInvalidOrValue, Identifier, IdentifierObjPair, BaseCategorizedRegistry, Logger

saoirse_id = "saoirse"

class SaoirseRegistry(BaseCategorizedRegistry):
    def __init__(self):
        super().__init__()

        self.category_tags = self.SaoirseRegistryCategories()

        self.register_items()
        self.register_tiles()
        self.register_entities()

    def register_items(self):
        self.register_category(self.category_tags.items)

    def register_tiles(self):
        self.register_category(self.category_tags.tiles)

    def register_entities(self):
        self.register_category(self.category_tags.entities)


    class SaoirseRegistryCategories():
        def __init__(self):
            self.items = "items"
            self.tiles = "tiles"
            self.entities = "entities"

registry = SaoirseRegistry()

Logger.log_msg(saoirse_id, registry.get_categories())
Logger.log_msg(saoirse_id, registry.get_entries())

class SaoriseDimension(BlankInvalidOrValue):
    def __init__(self):
        pass
```
