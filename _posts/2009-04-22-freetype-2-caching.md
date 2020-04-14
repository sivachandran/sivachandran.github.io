---
layout: post
title: "FreeType 2 Caching"
date: 2009-04-22 22:03:00 +0530
tags: [embedded, c-cpp]
---

Recently I found that FreeType 2 has support for caching. Previously I used to cache myself in custom data structure in order to avoid rendering glyph every time. But this new caching mechanism provides robust way of caching rendered glyph. It also helps us to cache other things like char map lookups, face sizes and etc. It also flushes the cache whenever there is no room for new one. Now I am going to explain some of component of FreeType 2 cache sub-system and how to use it. Please refer http://freetype.sourceforge.net/freetype2/docs/reference/ft2-cache_subsystem.html for more information.

## FTC_Manager

This is the main component in cache sub-system that is responsible for managing all kind of cache. We have to create a FTC_Manager before doing anything with cache sub-system. The function for creating a new FTC_Manager is
```c
FTC_Manager_New(FT_Library library,
                FT_UInt max_faces,
                FT_UInt max_sizes,
                FT_ULong max_bytes,
                FTC_Face_Requester requester,
                FT_Pointer req_data,
                FTC_Manager *amanager);
```
here `max_faces`, `max_sizes` determines the max no. of faces and sizes to cache. The `max_bytes` is the max amount of memory that can be used for caching. The `requester` and `req_data` is callback mechanism for converting face id into face. When we use the caching functions we don't pass FT_Face directly, instead we pass a face id which will be mapped into corresponding FT_Face by this callback function. The face id is typedef-ed as FT_Pointer so you can use any integer or pointer value as face id. If you are going to use only one face then you can simply return the FT_Face for any face id or if you are going to use more than one faces then probably you can pass the filename via `req_data` then load and return FT_Face inside `requester` callback. Note, the "requester" callback function will be called only once for the first lookup(Face, Size, CMap, Image, SBit) of the face id. When the "requester" callback returns a FT_Face for given face id the FTC_Manager stores the pair for future lookups. So don't expect the `requester` callback to be called for every lookup.

Once we done with caching we can call FTC_Manager_Done which will free the memory used for caching.

## FTC_Node

All objects(Face, Size, CMap, SBit and Image) in caching sub-system associated with FTC_Node. FTC_Manager manipulates all type of objects in the form of FTC_Node and each FTC_Node is reference counted. We have to manually de-reference(FTC_Node_Unref) the node when we no longer use it. All the caching lookup function accepts a pointer to FTC_Node, if we provide an FTC_Node pointer then the FTC_Node of the lookup result will be returned. The returned FTC_Node will have reference count incremented by one. So its our responsibility to de-referencing the FTC_Node when we no longer need it. We can also pass NULL as address of FTC_Node pointer if we don't want to manually manage the FTC_Node, in that case the lookup result which we get will be freed in the next lookup.

## FTC_SBitCache

A rendered glyph can be cached using FTC_SBitCache(SBit stands for Small Bitmap). First we need to create the SBitCache using FTC_SBitCacheNew, then use the returned FTC_SBitCache for any SBit lookup. The SBit lookup can be done using the function FTC_SBitCacheLookup

```
FTC_SBitCache_Lookup(FTC_SBitCache cache,
                     FTC_ImageType type,
                     FT_UInt gindex,
                     FTC_SBit *sbit,
                     FTC_Node *anode);
```
here the parameter "type" holds the face id, width, height and rendering flags(i.e. FT_LOAD_RENDER). "gindex" is the glyph index we want to render as bitmap and the resultant FTC_SBit is stored in "sbit". A FTC_SBit holds all the necessary information to paint the bitmap on a surface/canvas. The "buffer" field of FTC_SBit nothing but the "buffer" field of FT_Bitmap which holds the raw bitmap data.
## Sample
The following code snippet shows how typical usage of FreeType 2 Caching. I didn't show handling the return value of the FreeType 2 API to make code readable, but don't do that in your real code.
```c
int main(int argc, char **argv)
{
  FT_Library lib;
  FTC_Manager ftcManager;
  FTC_SBitCache ftcSBitCache;
  FTC_SBit ftcSBit;
  int ret;

  ret = FT_Init_FreeType(&amp;lib);
  assert(ret == 0);

  ret = FTC_Manager_New(lib, 1, 1, 1024 * 1024, ftcFaceRequester, fontFilename, &ftcManager);
  assert(ret == 0);
  ret = FTC_CMapCache_New(ftcManager, &ftcCMapCache);
  assert(ret == 0);
  ret = FTC_SBitCache_New(ftcManager, &ftcSBitCache);
  assert(ret == 0);

  ftcImageType.face_id = 0;
  ftcImageType.width = 16;
  ftcImageType.height = 16;
  ftcImageType.flags = FT_LOAD_DEFAULT | FT_LOAD_RENDER;

  glyphIndex = FTC_CMapCache_Lookup(ftcCMapCache, 0, 0, 'a');
  assert(glyphIndex > 0);
  ret = FTC_SBitCache_Lookup(ftcSBitCache, &ftcImageType, glyphIndex, &ftcSBit, &ftcNode);
  assert(ret == 0);

  /* render the glyph */
  FTC_Node_Unref(ftcNode, ftcManager);
  FTC_Manager_Done(ftcManager);
}

FT_Error FtcFaceRequester(FTC_FaceID faceID, FT_Library lib, FT_Pointer reqData, FT_Face *face)
{
  int ret = FT_New_Face(lib, (char *)reqData, 0, face);
  assert(ret == 0);
  return 0;
}
```