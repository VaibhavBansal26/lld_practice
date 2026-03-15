# 14. Music Streaming (Spotify)


Code

```python
from __future__ import annotations

from dataclasses import dataclass, field
from datetime import datetime, timedelta
from enum import Enum
from typing import Dict, List, Optional


# =========================
# Enums
# =========================

class SubscriptionType(str, Enum):
    FREE = "FREE"
    PREMIUM = "PREMIUM"


class SubscriptionStatus(str, Enum):
    ACTIVE = "ACTIVE"
    EXPIRED = "EXPIRED"
    CANCELLED = "CANCELLED"


class PlaybackState(str, Enum):
    PLAYING = "PLAYING"
    PAUSED = "PAUSED"
    STOPPED = "STOPPED"


# =========================
# Core Models
# =========================

@dataclass
class Subscription:
    subscription_type: SubscriptionType
    status: SubscriptionStatus
    start_date: datetime
    end_date: Optional[datetime] = None

    def is_active(self) -> bool:
        if self.status != SubscriptionStatus.ACTIVE:
            return False
        if self.end_date is None:
            return True
        return datetime.now() <= self.end_date


@dataclass(frozen=True)
class Artist:
    artist_id: str
    name: str


@dataclass(frozen=True)
class Song:
    song_id: str
    title: str
    duration_seconds: int
    artist: Artist
    album_id: Optional[str] = None


@dataclass
class Album:
    album_id: str
    title: str
    artist: Artist
    songs: List[Song] = field(default_factory=list)

    def add_song(self, song: Song) -> None:
        self.songs.append(song)


@dataclass
class Playlist:
    playlist_id: str
    name: str
    owner_user_id: str
    songs: List[Song] = field(default_factory=list)
    is_public: bool = False

    def add_song(self, song: Song) -> None:
        self.songs.append(song)

    def remove_song(self, song_id: str) -> None:
        self.songs = [song for song in self.songs if song.song_id != song_id]


# =========================
# Library / History
# =========================

@dataclass
class Library:
    liked_songs: List[Song] = field(default_factory=list)
    saved_albums: List[Album] = field(default_factory=list)
    followed_artists: List[Artist] = field(default_factory=list)

    def like_song(self, song: Song) -> None:
        if all(existing.song_id != song.song_id for existing in self.liked_songs):
            self.liked_songs.append(song)

    def unlike_song(self, song_id: str) -> None:
        self.liked_songs = [song for song in self.liked_songs if song.song_id != song_id]

    def save_album(self, album: Album) -> None:
        if all(existing.album_id != album.album_id for existing in self.saved_albums):
            self.saved_albums.append(album)

    def follow_artist(self, artist: Artist) -> None:
        if all(existing.artist_id != artist.artist_id for existing in self.followed_artists):
            self.followed_artists.append(artist)


@dataclass
class PlayHistoryItem:
    song: Song
    played_at: datetime


@dataclass
class PlayHistory:
    items: List[PlayHistoryItem] = field(default_factory=list)

    def add_play(self, song: Song) -> None:
        self.items.append(PlayHistoryItem(song=song, played_at=datetime.now()))

    def get_recent(self, limit: int = 10) -> List[PlayHistoryItem]:
        return self.items[-limit:][::-1]


# =========================
# Queue / Playback Session
# =========================

@dataclass
class PlayQueue:
    songs: List[Song] = field(default_factory=list)
    current_index: int = -1

    def load(self, songs: List[Song], start_index: int = 0) -> None:
        self.songs = songs[:]
        self.current_index = start_index if songs and 0 <= start_index < len(songs) else -1

    def add_to_queue(self, song: Song) -> None:
        self.songs.append(song)
        if self.current_index == -1:
            self.current_index = 0

    def get_current_song(self) -> Optional[Song]:
        if 0 <= self.current_index < len(self.songs):
            return self.songs[self.current_index]
        return None

    def next_song(self) -> Optional[Song]:
        if self.current_index + 1 < len(self.songs):
            self.current_index += 1
            return self.songs[self.current_index]
        return None

    def previous_song(self) -> Optional[Song]:
        if self.current_index - 1 >= 0:
            self.current_index -= 1
            return self.songs[self.current_index]
        return None

    def clear(self) -> None:
        self.songs.clear()
        self.current_index = -1


@dataclass
class PlaybackSession:
    user_id: str
    queue: PlayQueue = field(default_factory=PlayQueue)
    state: PlaybackState = PlaybackState.STOPPED
    current_position_seconds: int = 0

    def play(self) -> None:
        if self.queue.get_current_song() is not None:
            self.state = PlaybackState.PLAYING

    def pause(self) -> None:
        if self.state == PlaybackState.PLAYING:
            self.state = PlaybackState.PAUSED

    def stop(self) -> None:
        self.state = PlaybackState.STOPPED
        self.current_position_seconds = 0

    def resume(self) -> None:
        if self.state == PlaybackState.PAUSED:
            self.state = PlaybackState.PLAYING


# =========================
# User
# =========================

@dataclass
class User:
    user_id: str
    username: str
    subscription: Subscription
    library: Library = field(default_factory=Library)
    playlists: List[Playlist] = field(default_factory=list)
    play_history: PlayHistory = field(default_factory=PlayHistory)

    def create_playlist(self, playlist_id: str, name: str, is_public: bool = False) -> Playlist:
        playlist = Playlist(
            playlist_id=playlist_id,
            name=name,
            owner_user_id=self.user_id,
            is_public=is_public,
        )
        self.playlists.append(playlist)
        return playlist

    def get_playlist(self, playlist_id: str) -> Optional[Playlist]:
        for playlist in self.playlists:
            if playlist.playlist_id == playlist_id:
                return playlist
        return None


# =========================
# Catalog
# =========================

class Catalog:
    def __init__(self) -> None:
        self.songs: Dict[str, Song] = {}
        self.albums: Dict[str, Album] = {}
        self.artists: Dict[str, Artist] = {}

    def add_artist(self, artist: Artist) -> None:
        self.artists[artist.artist_id] = artist

    def add_song(self, song: Song) -> None:
        self.songs[song.song_id] = song

    def add_album(self, album: Album) -> None:
        self.albums[album.album_id] = album

    def get_song(self, song_id: str) -> Optional[Song]:
        return self.songs.get(song_id)

    def get_album(self, album_id: str) -> Optional[Album]:
        return self.albums.get(album_id)

    def get_artist(self, artist_id: str) -> Optional[Artist]:
        return self.artists.get(artist_id)

    def search_songs(self, query: str) -> List[Song]:
        q = query.lower()
        return [song for song in self.songs.values() if q in song.title.lower()]

    def search_artists(self, query: str) -> List[Artist]:
        q = query.lower()
        return [artist for artist in self.artists.values() if q in artist.name.lower()]

    def search_albums(self, query: str) -> List[Album]:
        q = query.lower()
        return [album for album in self.albums.values() if q in album.title.lower()]


# =========================
# Music Streaming Service
# =========================

class MusicStreamingService:
    def __init__(self, catalog: Catalog) -> None:
        self.catalog = catalog
        self.users: Dict[str, User] = {}
        self.sessions: Dict[str, PlaybackSession] = {}

    def add_user(self, user: User) -> None:
        self.users[user.user_id] = user
        self.sessions[user.user_id] = PlaybackSession(user_id=user.user_id)

    def get_user(self, user_id: str) -> Optional[User]:
        return self.users.get(user_id)

    def get_session(self, user_id: str) -> PlaybackSession:
        if user_id not in self.sessions:
            raise ValueError("User session not found")
        return self.sessions[user_id]

    def _validate_user_can_play(self, user: User) -> None:
        if not user.subscription.is_active():
            raise ValueError("Subscription is not active")

    def play_song(self, user_id: str, song_id: str) -> None:
        user = self.users[user_id]
        self._validate_user_can_play(user)

        song = self.catalog.get_song(song_id)
        if song is None:
            raise ValueError("Song not found")

        session = self.sessions[user_id]
        session.queue.load([song], 0)
        session.current_position_seconds = 0
        session.play()
        user.play_history.add_play(song)

    def play_album(self, user_id: str, album_id: str, start_index: int = 0) -> None:
        user = self.users[user_id]
        self._validate_user_can_play(user)

        album = self.catalog.get_album(album_id)
        if album is None:
            raise ValueError("Album not found")
        if not album.songs:
            raise ValueError("Album has no songs")

        session = self.sessions[user_id]
        session.queue.load(album.songs, start_index)
        session.current_position_seconds = 0
        session.play()

        current_song = session.queue.get_current_song()
        if current_song:
            user.play_history.add_play(current_song)

    def play_playlist(self, user_id: str, playlist_id: str, start_index: int = 0) -> None:
        user = self.users[user_id]
        self._validate_user_can_play(user)

        playlist = user.get_playlist(playlist_id)
        if playlist is None:
            raise ValueError("Playlist not found")
        if not playlist.songs:
            raise ValueError("Playlist is empty")

        session = self.sessions[user_id]
        session.queue.load(playlist.songs, start_index)
        session.current_position_seconds = 0
        session.play()

        current_song = session.queue.get_current_song()
        if current_song:
            user.play_history.add_play(current_song)

    def pause(self, user_id: str) -> None:
        self.sessions[user_id].pause()

    def resume(self, user_id: str) -> None:
        user = self.users[user_id]
        self._validate_user_can_play(user)
        self.sessions[user_id].resume()

    def stop(self, user_id: str) -> None:
        self.sessions[user_id].stop()

    def next_song(self, user_id: str) -> Optional[Song]:
        user = self.users[user_id]
        session = self.sessions[user_id]
        next_song = session.queue.next_song()
        session.current_position_seconds = 0

        if next_song:
            session.play()
            user.play_history.add_play(next_song)
        else:
            session.stop()

        return next_song

    def previous_song(self, user_id: str) -> Optional[Song]:
        session = self.sessions[user_id]
        song = session.queue.previous_song()
        session.current_position_seconds = 0
        if song:
            session.play()
        return song

    def add_song_to_queue(self, user_id: str, song_id: str) -> None:
        song = self.catalog.get_song(song_id)
        if song is None:
            raise ValueError("Song not found")
        self.sessions[user_id].queue.add_to_queue(song)

    def add_song_to_playlist(self, user_id: str, playlist_id: str, song_id: str) -> None:
        user = self.users[user_id]
        playlist = user.get_playlist(playlist_id)
        if playlist is None:
            raise ValueError("Playlist not found")

        song = self.catalog.get_song(song_id)
        if song is None:
            raise ValueError("Song not found")

        playlist.add_song(song)

    def remove_song_from_playlist(self, user_id: str, playlist_id: str, song_id: str) -> None:
        user = self.users[user_id]
        playlist = user.get_playlist(playlist_id)
        if playlist is None:
            raise ValueError("Playlist not found")
        playlist.remove_song(song_id)

    def like_song(self, user_id: str, song_id: str) -> None:
        user = self.users[user_id]
        song = self.catalog.get_song(song_id)
        if song is None:
            raise ValueError("Song not found")
        user.library.like_song(song)

    def unlike_song(self, user_id: str, song_id: str) -> None:
        user = self.users[user_id]
        user.library.unlike_song(song_id)

    def save_album(self, user_id: str, album_id: str) -> None:
        user = self.users[user_id]
        album = self.catalog.get_album(album_id)
        if album is None:
            raise ValueError("Album not found")
        user.library.save_album(album)

    def follow_artist(self, user_id: str, artist_id: str) -> None:
        user = self.users[user_id]
        artist = self.catalog.get_artist(artist_id)
        if artist is None:
            raise ValueError("Artist not found")
        user.library.follow_artist(artist)

    def search(self, query: str) -> Dict[str, List]:
        return {
            "songs": self.catalog.search_songs(query),
            "artists": self.catalog.search_artists(query),
            "albums": self.catalog.search_albums(query),
        }


# =========================
# Example Usage
# =========================

if __name__ == "__main__":
    # Artists
    artist1 = Artist(artist_id="a1", name="The Weeknd")
    artist2 = Artist(artist_id="a2", name="Daft Punk")

    # Songs
    song1 = Song(
        song_id="s1",
        title="Blinding Lights",
        duration_seconds=200,
        artist=artist1,
        album_id="al1",
    )
    song2 = Song(
        song_id="s2",
        title="Save Your Tears",
        duration_seconds=215,
        artist=artist1,
        album_id="al1",
    )
    song3 = Song(
        song_id="s3",
        title="Starboy",
        duration_seconds=230,
        artist=artist1,
        album_id="al2",
    )

    # Albums
    album1 = Album(album_id="al1", title="After Hours", artist=artist1)
    album1.add_song(song1)
    album1.add_song(song2)

    album2 = Album(album_id="al2", title="Starboy", artist=artist1)
    album2.add_song(song3)

    # Catalog
    catalog = Catalog()
    catalog.add_artist(artist1)
    catalog.add_artist(artist2)
    catalog.add_song(song1)
    catalog.add_song(song2)
    catalog.add_song(song3)
    catalog.add_album(album1)
    catalog.add_album(album2)

    # User
    subscription = Subscription(
        subscription_type=SubscriptionType.PREMIUM,
        status=SubscriptionStatus.ACTIVE,
        start_date=datetime.now(),
        end_date=datetime.now() + timedelta(days=30),
    )

    user = User(user_id="u1", username="vaibhav", subscription=subscription)
    playlist = user.create_playlist("p1", "Favorites")
    playlist.add_song(song1)
    playlist.add_song(song2)

    # Service
    service = MusicStreamingService(catalog)
    service.add_user(user)

    # Play a single song
    service.play_song("u1", "s1")
    print("Current song:", service.get_session("u1").queue.get_current_song())

    # Queue another song
    service.add_song_to_queue("u1", "s2")
    print("Queue:", [song.title for song in service.get_session("u1").queue.songs])

    # Next song
    next_song = service.next_song("u1")
    print("Next song:", next_song.title if next_song else None)

    # Play playlist
    service.play_playlist("u1", "p1")
    print("Playlist current song:", service.get_session("u1").queue.get_current_song())

    # Like song
    service.like_song("u1", "s3")
    print("Liked songs:", [song.title for song in user.library.liked_songs])

    # Save album
    service.save_album("u1", "al1")
    print("Saved albums:", [album.title for album in user.library.saved_albums])

    # Follow artist
    service.follow_artist("u1", "a1")
    print("Followed artists:", [artist.name for artist in user.library.followed_artists])

    # Search
    result = service.search("save")
    print("Search songs:", [song.title for song in result["songs"]])

    # Play history
    print("Recently played:", [item.song.title for item in user.play_history.get_recent()])
```

DEEP DIVE

```python
from __future__ import annotations

from abc import ABC, abstractmethod
from dataclasses import dataclass, field
from datetime import datetime, timedelta
from enum import Enum
from typing import Dict, List, Optional, Set
import random


# =========================
# Enums
# =========================

class SubscriptionType(str, Enum):
    FREE = "FREE"
    PREMIUM = "PREMIUM"


class SubscriptionStatus(str, Enum):
    ACTIVE = "ACTIVE"
    EXPIRED = "EXPIRED"
    CANCELLED = "CANCELLED"


class PlaybackState(str, Enum):
    PLAYING = "PLAYING"
    PAUSED = "PAUSED"
    STOPPED = "STOPPED"


class RepeatMode(str, Enum):
    OFF = "OFF"
    ONE = "ONE"
    ALL = "ALL"


class DeviceType(str, Enum):
    MOBILE = "MOBILE"
    TABLET = "TABLET"
    WEB = "WEB"
    DESKTOP = "DESKTOP"
    SPEAKER = "SPEAKER"
    TV = "TV"


class PlaylistRole(str, Enum):
    OWNER = "OWNER"
    COLLABORATOR = "COLLABORATOR"
    VIEWER = "VIEWER"


# =========================
# Core Models
# =========================

@dataclass
class Subscription:
    subscription_type: SubscriptionType
    status: SubscriptionStatus
    start_date: datetime
    end_date: Optional[datetime] = None

    def is_active(self) -> bool:
        if self.status != SubscriptionStatus.ACTIVE:
            return False
        if self.end_date is None:
            return True
        return datetime.now() <= self.end_date


@dataclass(frozen=True)
class Artist:
    artist_id: str
    name: str


@dataclass(frozen=True)
class Song:
    song_id: str
    title: str
    duration_seconds: int
    artist: Artist
    album_id: Optional[str] = None


@dataclass
class Album:
    album_id: str
    title: str
    artist: Artist
    songs: List[Song] = field(default_factory=list)

    def add_song(self, song: Song) -> None:
        self.songs.append(song)


@dataclass(frozen=True)
class Device:
    device_id: str
    name: str
    device_type: DeviceType


@dataclass
class Playlist:
    playlist_id: str
    name: str
    owner_user_id: str
    songs: List[Song] = field(default_factory=list)
    is_public: bool = False
    collaborators: Dict[str, PlaylistRole] = field(default_factory=dict)

    def __post_init__(self) -> None:
        self.collaborators[self.owner_user_id] = PlaylistRole.OWNER

    def can_edit(self, user_id: str) -> bool:
        return self.collaborators.get(user_id) in {PlaylistRole.OWNER, PlaylistRole.COLLABORATOR}

    def add_collaborator(self, user_id: str, role: PlaylistRole) -> None:
        self.collaborators[user_id] = role

    def add_song(self, user_id: str, song: Song) -> None:
        if not self.can_edit(user_id):
            raise PermissionError("User cannot edit playlist")
        self.songs.append(song)

    def remove_song(self, user_id: str, song_id: str) -> None:
        if not self.can_edit(user_id):
            raise PermissionError("User cannot edit playlist")
        self.songs = [song for song in self.songs if song.song_id != song_id]


# =========================
# Library / History / Downloads
# =========================

@dataclass
class Library:
    liked_songs: List[Song] = field(default_factory=list)
    saved_albums: List[Album] = field(default_factory=list)
    followed_artists: List[Artist] = field(default_factory=list)

    def like_song(self, song: Song) -> None:
        if all(existing.song_id != song.song_id for existing in self.liked_songs):
            self.liked_songs.append(song)

    def unlike_song(self, song_id: str) -> None:
        self.liked_songs = [song for song in self.liked_songs if song.song_id != song_id]

    def save_album(self, album: Album) -> None:
        if all(existing.album_id != album.album_id for existing in self.saved_albums):
            self.saved_albums.append(album)

    def follow_artist(self, artist: Artist) -> None:
        if all(existing.artist_id != artist.artist_id for existing in self.followed_artists):
            self.followed_artists.append(artist)


@dataclass
class PlayHistoryItem:
    song: Song
    played_at: datetime
    device_id: Optional[str] = None


@dataclass
class PlayHistory:
    items: List[PlayHistoryItem] = field(default_factory=list)

    def add_play(self, song: Song, device_id: Optional[str] = None) -> None:
        self.items.append(
            PlayHistoryItem(song=song, played_at=datetime.now(), device_id=device_id)
        )

    def get_recent(self, limit: int = 10) -> List[PlayHistoryItem]:
        return self.items[-limit:][::-1]


@dataclass
class DownloadedSong:
    song: Song
    downloaded_at: datetime


@dataclass
class DownloadLibrary:
    songs: Dict[str, DownloadedSong] = field(default_factory=dict)

    def add_song(self, song: Song) -> None:
        self.songs[song.song_id] = DownloadedSong(song=song, downloaded_at=datetime.now())

    def has_song(self, song_id: str) -> bool:
        return song_id in self.songs

    def remove_song(self, song_id: str) -> None:
        self.songs.pop(song_id, None)


# =========================
# Queue / Playback
# =========================

@dataclass
class PlayQueue:
    songs: List[Song] = field(default_factory=list)
    current_index: int = -1
    shuffle_enabled: bool = False
    repeat_mode: RepeatMode = RepeatMode.OFF

    def load(self, songs: List[Song], start_index: int = 0, shuffle_enabled: bool = False) -> None:
        self.songs = songs[:]
        self.shuffle_enabled = shuffle_enabled
        if shuffle_enabled and self.songs:
            current_song = self.songs[start_index] if 0 <= start_index < len(self.songs) else self.songs[0]
            remaining = [song for i, song in enumerate(self.songs) if i != start_index]
            random.shuffle(remaining)
            self.songs = [current_song] + remaining
            self.current_index = 0
        else:
            self.current_index = start_index if songs and 0 <= start_index < len(songs) else -1

    def add_to_queue(self, song: Song) -> None:
        self.songs.append(song)
        if self.current_index == -1:
            self.current_index = 0

    def get_current_song(self) -> Optional[Song]:
        if 0 <= self.current_index < len(self.songs):
            return self.songs[self.current_index]
        return None

    def next_song(self) -> Optional[Song]:
        if not self.songs:
            return None

        if self.repeat_mode == RepeatMode.ONE:
            return self.get_current_song()

        if self.current_index + 1 < len(self.songs):
            self.current_index += 1
            return self.songs[self.current_index]

        if self.repeat_mode == RepeatMode.ALL:
            self.current_index = 0
            return self.songs[self.current_index]

        return None

    def previous_song(self) -> Optional[Song]:
        if not self.songs:
            return None

        if self.current_index - 1 >= 0:
            self.current_index -= 1
            return self.songs[self.current_index]

        if self.repeat_mode == RepeatMode.ALL and self.songs:
            self.current_index = len(self.songs) - 1
            return self.songs[self.current_index]

        return None

    def clear(self) -> None:
        self.songs.clear()
        self.current_index = -1


@dataclass
class PlaybackSession:
    user_id: str
    device: Device
    queue: PlayQueue = field(default_factory=PlayQueue)
    state: PlaybackState = PlaybackState.STOPPED
    current_position_seconds: int = 0

    def play(self) -> None:
        if self.queue.get_current_song() is not None:
            self.state = PlaybackState.PLAYING

    def pause(self) -> None:
        if self.state == PlaybackState.PLAYING:
            self.state = PlaybackState.PAUSED

    def stop(self) -> None:
        self.state = PlaybackState.STOPPED
        self.current_position_seconds = 0

    def resume(self) -> None:
        if self.state == PlaybackState.PAUSED:
            self.state = PlaybackState.PLAYING


# =========================
# User
# =========================

@dataclass
class User:
    user_id: str
    username: str
    subscription: Subscription
    library: Library = field(default_factory=Library)
    playlists: List[Playlist] = field(default_factory=list)
    play_history: PlayHistory = field(default_factory=PlayHistory)
    downloads: DownloadLibrary = field(default_factory=DownloadLibrary)
    devices: List[Device] = field(default_factory=list)

    def add_device(self, device: Device) -> None:
        if all(existing.device_id != device.device_id for existing in self.devices):
            self.devices.append(device)

    def create_playlist(self, playlist_id: str, name: str, is_public: bool = False) -> Playlist:
        playlist = Playlist(
            playlist_id=playlist_id,
            name=name,
            owner_user_id=self.user_id,
            is_public=is_public,
        )
        self.playlists.append(playlist)
        return playlist

    def get_playlist(self, playlist_id: str) -> Optional[Playlist]:
        for playlist in self.playlists:
            if playlist.playlist_id == playlist_id:
                return playlist
        return None


# =========================
# Catalog
# =========================

class Catalog:
    def __init__(self) -> None:
        self.songs: Dict[str, Song] = {}
        self.albums: Dict[str, Album] = {}
        self.artists: Dict[str, Artist] = {}

    def add_artist(self, artist: Artist) -> None:
        self.artists[artist.artist_id] = artist

    def add_song(self, song: Song) -> None:
        self.songs[song.song_id] = song

    def add_album(self, album: Album) -> None:
        self.albums[album.album_id] = album

    def get_song(self, song_id: str) -> Optional[Song]:
        return self.songs.get(song_id)

    def get_album(self, album_id: str) -> Optional[Album]:
        return self.albums.get(album_id)

    def get_artist(self, artist_id: str) -> Optional[Artist]:
        return self.artists.get(artist_id)

    def search_songs(self, query: str) -> List[Song]:
        q = query.lower()
        return [song for song in self.songs.values() if q in song.title.lower()]

    def search_artists(self, query: str) -> List[Artist]:
        q = query.lower()
        return [artist for artist in self.artists.values() if q in artist.name.lower()]

    def search_albums(self, query: str) -> List[Album]:
        q = query.lower()
        return [album for album in self.albums.values() if q in album.title.lower()]


# =========================
# Playback Policy Strategy
# =========================

class PlaybackPolicy(ABC):
    @abstractmethod
    def validate_play(self, user: User, active_sessions: List[PlaybackSession]) -> None:
        pass

    @abstractmethod
    def can_download(self) -> bool:
        pass

    @abstractmethod
    def max_concurrent_streams(self) -> int:
        pass


class FreePlaybackPolicy(PlaybackPolicy):
    def validate_play(self, user: User, active_sessions: List[PlaybackSession]) -> None:
        if not user.subscription.is_active():
            raise ValueError("Subscription is not active")
        if len(active_sessions) >= 1:
            raise ValueError("Free plan supports only one active stream")

    def can_download(self) -> bool:
        return False

    def max_concurrent_streams(self) -> int:
        return 1


class PremiumPlaybackPolicy(PlaybackPolicy):
    def validate_play(self, user: User, active_sessions: List[PlaybackSession]) -> None:
        if not user.subscription.is_active():
            raise ValueError("Subscription is not active")
        if len(active_sessions) >= 3:
            raise ValueError("Premium concurrent stream limit reached")

    def can_download(self) -> bool:
        return True

    def max_concurrent_streams(self) -> int:
        return 3


# =========================
# Service
# =========================

class MusicStreamingService:
    def __init__(self, catalog: Catalog) -> None:
        self.catalog = catalog
        self.users: Dict[str, User] = {}
        self.sessions_by_user: Dict[str, Dict[str, PlaybackSession]] = {}

    def add_user(self, user: User) -> None:
        self.users[user.user_id] = user
        self.sessions_by_user[user.user_id] = {}

    def get_user(self, user_id: str) -> Optional[User]:
        return self.users.get(user_id)

    def _get_policy(self, user: User) -> PlaybackPolicy:
        if user.subscription.subscription_type == SubscriptionType.PREMIUM:
            return PremiumPlaybackPolicy()
        return FreePlaybackPolicy()

    def _get_user_sessions(self, user_id: str) -> Dict[str, PlaybackSession]:
        if user_id not in self.sessions_by_user:
            raise ValueError("User not found")
        return self.sessions_by_user[user_id]

    def _get_or_create_session(self, user: User, device: Device) -> PlaybackSession:
        sessions = self._get_user_sessions(user.user_id)
        if device.device_id not in sessions:
            sessions[device.device_id] = PlaybackSession(user_id=user.user_id, device=device)
        return sessions[device.device_id]

    def _get_active_sessions(self, user_id: str) -> List[PlaybackSession]:
        sessions = self._get_user_sessions(user_id)
        return [s for s in sessions.values() if s.state in {PlaybackState.PLAYING, PlaybackState.PAUSED}]

    def register_device(self, user_id: str, device: Device) -> None:
        user = self.users[user_id]
        user.add_device(device)

    def play_song(self, user_id: str, device: Device, song_id: str) -> None:
        user = self.users[user_id]
        policy = self._get_policy(user)
        active_sessions = self._get_active_sessions(user_id)
        policy.validate_play(user, [s for s in active_sessions if s.device.device_id != device.device_id])

        song = self.catalog.get_song(song_id)
        if song is None:
            raise ValueError("Song not found")

        session = self._get_or_create_session(user, device)
        session.queue.load([song], 0)
        session.current_position_seconds = 0
        session.play()
        user.play_history.add_play(song, device.device_id)

    def play_album(
        self,
        user_id: str,
        device: Device,
        album_id: str,
        start_index: int = 0,
        shuffle_enabled: bool = False,
    ) -> None:
        user = self.users[user_id]
        policy = self._get_policy(user)
        active_sessions = self._get_active_sessions(user_id)
        policy.validate_play(user, [s for s in active_sessions if s.device.device_id != device.device_id])

        album = self.catalog.get_album(album_id)
        if album is None:
            raise ValueError("Album not found")
        if not album.songs:
            raise ValueError("Album has no songs")

        session = self._get_or_create_session(user, device)
        session.queue.load(album.songs, start_index, shuffle_enabled=shuffle_enabled)
        session.current_position_seconds = 0
        session.play()

        current_song = session.queue.get_current_song()
        if current_song:
            user.play_history.add_play(current_song, device.device_id)

    def play_playlist(
        self,
        user_id: str,
        device: Device,
        playlist_id: str,
        start_index: int = 0,
        shuffle_enabled: bool = False,
    ) -> None:
        user = self.users[user_id]
        policy = self._get_policy(user)
        active_sessions = self._get_active_sessions(user_id)
        policy.validate_play(user, [s for s in active_sessions if s.device.device_id != device.device_id])

        playlist = user.get_playlist(playlist_id)
        if playlist is None:
            raise ValueError("Playlist not found")
        if not playlist.songs:
            raise ValueError("Playlist is empty")

        session = self._get_or_create_session(user, device)
        session.queue.load(playlist.songs, start_index, shuffle_enabled=shuffle_enabled)
        session.current_position_seconds = 0
        session.play()

        current_song = session.queue.get_current_song()
        if current_song:
            user.play_history.add_play(current_song, device.device_id)

    def pause(self, user_id: str, device_id: str) -> None:
        self._get_user_sessions(user_id)[device_id].pause()

    def resume(self, user_id: str, device_id: str) -> None:
        self._get_user_sessions(user_id)[device_id].resume()

    def stop(self, user_id: str, device_id: str) -> None:
        self._get_user_sessions(user_id)[device_id].stop()

    def next_song(self, user_id: str, device_id: str) -> Optional[Song]:
        user = self.users[user_id]
        session = self._get_user_sessions(user_id)[device_id]
        next_song = session.queue.next_song()
        session.current_position_seconds = 0

        if next_song:
            session.play()
            user.play_history.add_play(next_song, device_id)
        else:
            session.stop()

        return next_song

    def previous_song(self, user_id: str, device_id: str) -> Optional[Song]:
        session = self._get_user_sessions(user_id)[device_id]
        song = session.queue.previous_song()
        session.current_position_seconds = 0
        if song:
            session.play()
        return song

    def set_repeat_mode(self, user_id: str, device_id: str, repeat_mode: RepeatMode) -> None:
        self._get_user_sessions(user_id)[device_id].queue.repeat_mode = repeat_mode

    def add_song_to_queue(self, user_id: str, device_id: str, song_id: str) -> None:
        song = self.catalog.get_song(song_id)
        if song is None:
            raise ValueError("Song not found")
        self._get_user_sessions(user_id)[device_id].queue.add_to_queue(song)

    def add_song_to_playlist(self, acting_user_id: str, playlist_id: str, song_id: str) -> None:
        user = self.users[acting_user_id]
        playlist = user.get_playlist(playlist_id)
        if playlist is None:
            raise ValueError("Playlist not found")

        song = self.catalog.get_song(song_id)
        if song is None:
            raise ValueError("Song not found")

        playlist.add_song(acting_user_id, song)

    def remove_song_from_playlist(self, acting_user_id: str, playlist_id: str, song_id: str) -> None:
        user = self.users[acting_user_id]
        playlist = user.get_playlist(playlist_id)
        if playlist is None:
            raise ValueError("Playlist not found")
        playlist.remove_song(acting_user_id, song_id)

    def add_collaborator(self, owner_user_id: str, playlist_id: str, collaborator_user_id: str) -> None:
        user = self.users[owner_user_id]
        playlist = user.get_playlist(playlist_id)
        if playlist is None:
            raise ValueError("Playlist not found")
        if playlist.owner_user_id != owner_user_id:
            raise PermissionError("Only owner can add collaborators")

        playlist.add_collaborator(collaborator_user_id, PlaylistRole.COLLABORATOR)

    def like_song(self, user_id: str, song_id: str) -> None:
        user = self.users[user_id]
        song = self.catalog.get_song(song_id)
        if song is None:
            raise ValueError("Song not found")
        user.library.like_song(song)

    def unlike_song(self, user_id: str, song_id: str) -> None:
        self.users[user_id].library.unlike_song(song_id)

    def save_album(self, user_id: str, album_id: str) -> None:
        user = self.users[user_id]
        album = self.catalog.get_album(album_id)
        if album is None:
            raise ValueError("Album not found")
        user.library.save_album(album)

    def follow_artist(self, user_id: str, artist_id: str) -> None:
        user = self.users[user_id]
        artist = self.catalog.get_artist(artist_id)
        if artist is None:
            raise ValueError("Artist not found")
        user.library.follow_artist(artist)

    def download_song(self, user_id: str, song_id: str) -> None:
        user = self.users[user_id]
        policy = self._get_policy(user)
        if not policy.can_download():
            raise ValueError("Downloads are available only for premium users")

        song = self.catalog.get_song(song_id)
        if song is None:
            raise ValueError("Song not found")

        user.downloads.add_song(song)

    def play_downloaded_song(self, user_id: str, device: Device, song_id: str) -> None:
        user = self.users[user_id]
        if not user.downloads.has_song(song_id):
            raise ValueError("Song is not downloaded")

        song = self.catalog.get_song(song_id)
        if song is None:
            raise ValueError("Song metadata not found")

        session = self._get_or_create_session(user, device)
        session.queue.load([song], 0)
        session.current_position_seconds = 0
        session.play()
        user.play_history.add_play(song, device.device_id)

    def search(self, query: str) -> Dict[str, List]:
        return {
            "songs": self.catalog.search_songs(query),
            "artists": self.catalog.search_artists(query),
            "albums": self.catalog.search_albums(query),
        }


# =========================
# Example Usage
# =========================

if __name__ == "__main__":
    # artists
    artist = Artist(artist_id="a1", name="The Weeknd")

    # songs
    song1 = Song(song_id="s1", title="Blinding Lights", duration_seconds=200, artist=artist, album_id="al1")
    song2 = Song(song_id="s2", title="Save Your Tears", duration_seconds=215, artist=artist, album_id="al1")
    song3 = Song(song_id="s3", title="In Your Eyes", duration_seconds=220, artist=artist, album_id="al1")

    # album
    album = Album(album_id="al1", title="After Hours", artist=artist)
    album.add_song(song1)
    album.add_song(song2)
    album.add_song(song3)

    # catalog
    catalog = Catalog()
    catalog.add_artist(artist)
    catalog.add_song(song1)
    catalog.add_song(song2)
    catalog.add_song(song3)
    catalog.add_album(album)

    # user
    subscription = Subscription(
        subscription_type=SubscriptionType.PREMIUM,
        status=SubscriptionStatus.ACTIVE,
        start_date=datetime.now(),
        end_date=datetime.now() + timedelta(days=30),
    )
    user = User(user_id="u1", username="vaibhav", subscription=subscription)

    device1 = Device(device_id="d1", name="iPhone", device_type=DeviceType.MOBILE)
    device2 = Device(device_id="d2", name="MacBook", device_type=DeviceType.DESKTOP)
    user.add_device(device1)
    user.add_device(device2)

    playlist = user.create_playlist("p1", "Favorites", is_public=True)
    playlist.add_song("u1", song1)
    playlist.add_song("u1", song2)

    service = MusicStreamingService(catalog)
    service.add_user(user)

    # playback
    service.play_song("u1", device1, "s1")
    print("Current song on device1:", service._get_user_sessions("u1")["d1"].queue.get_current_song())

    service.play_album("u1", device2, "al1", shuffle_enabled=True)
    print("Current song on device2:", service._get_user_sessions("u1")["d2"].queue.get_current_song())

    service.set_repeat_mode("u1", "d2", RepeatMode.ALL)
    print("Repeat mode on device2:", service._get_user_sessions("u1")["d2"].queue.repeat_mode)

    # queue
    service.add_song_to_queue("u1", "d1", "s3")
    print("Queue device1:", [song.title for song in service._get_user_sessions("u1")["d1"].queue.songs])

    # library
    service.like_song("u1", "s2")
    service.save_album("u1", "al1")
    service.follow_artist("u1", "a1")
    print("Liked songs:", [song.title for song in user.library.liked_songs])

    # downloads
    service.download_song("u1", "s3")
    print("Downloaded songs:", list(user.downloads.songs.keys()))

    # history
    print("Recent plays:", [item.song.title for item in user.play_history.get_recent()])

    # search
    result = service.search("save")
    print("Search songs:", [song.title for song in result["songs"]])
```

global playlist registry

```python
from __future__ import annotations

from abc import ABC, abstractmethod
from dataclasses import dataclass, field
from datetime import datetime, timedelta
from enum import Enum
from typing import Dict, List, Optional, Set
import random


# =========================
# Enums
# =========================

class SubscriptionType(str, Enum):
    FREE = "FREE"
    PREMIUM = "PREMIUM"


class SubscriptionStatus(str, Enum):
    ACTIVE = "ACTIVE"
    EXPIRED = "EXPIRED"
    CANCELLED = "CANCELLED"


class PlaybackState(str, Enum):
    PLAYING = "PLAYING"
    PAUSED = "PAUSED"
    STOPPED = "STOPPED"


class RepeatMode(str, Enum):
    OFF = "OFF"
    ONE = "ONE"
    ALL = "ALL"


class DeviceType(str, Enum):
    MOBILE = "MOBILE"
    TABLET = "TABLET"
    WEB = "WEB"
    DESKTOP = "DESKTOP"
    SPEAKER = "SPEAKER"
    TV = "TV"


class PlaylistRole(str, Enum):
    OWNER = "OWNER"
    COLLABORATOR = "COLLABORATOR"
    VIEWER = "VIEWER"


# =========================
# Core Models
# =========================

@dataclass
class Subscription:
    subscription_type: SubscriptionType
    status: SubscriptionStatus
    start_date: datetime
    end_date: Optional[datetime] = None

    def is_active(self) -> bool:
        if self.status != SubscriptionStatus.ACTIVE:
            return False
        if self.end_date is None:
            return True
        return datetime.now() <= self.end_date


@dataclass(frozen=True)
class Artist:
    artist_id: str
    name: str


@dataclass(frozen=True)
class Song:
    song_id: str
    title: str
    duration_seconds: int
    artist: Artist
    album_id: Optional[str] = None


@dataclass
class Album:
    album_id: str
    title: str
    artist: Artist
    songs: List[Song] = field(default_factory=list)

    def add_song(self, song: Song) -> None:
        self.songs.append(song)


@dataclass(frozen=True)
class Device:
    device_id: str
    name: str
    device_type: DeviceType


@dataclass
class Playlist:
    playlist_id: str
    name: str
    owner_user_id: str
    songs: List[Song] = field(default_factory=list)
    is_public: bool = False
    collaborators: Dict[str, PlaylistRole] = field(default_factory=dict)

    def __post_init__(self) -> None:
        self.collaborators[self.owner_user_id] = PlaylistRole.OWNER

    def can_edit(self, user_id: str) -> bool:
        return self.collaborators.get(user_id) in {
            PlaylistRole.OWNER,
            PlaylistRole.COLLABORATOR,
        }

    def can_view(self, user_id: str) -> bool:
        return self.is_public or user_id in self.collaborators

    def add_collaborator(self, user_id: str, role: PlaylistRole) -> None:
        self.collaborators[user_id] = role

    def add_song(self, user_id: str, song: Song) -> None:
        if not self.can_edit(user_id):
            raise PermissionError("User cannot edit playlist")
        self.songs.append(song)

    def remove_song(self, user_id: str, song_id: str) -> None:
        if not self.can_edit(user_id):
            raise PermissionError("User cannot edit playlist")
        self.songs = [song for song in self.songs if song.song_id != song_id]


# =========================
# Library / History / Downloads
# =========================

@dataclass
class Library:
    liked_songs: List[Song] = field(default_factory=list)
    saved_albums: List[Album] = field(default_factory=list)
    followed_artists: List[Artist] = field(default_factory=list)

    def like_song(self, song: Song) -> None:
        if all(existing.song_id != song.song_id for existing in self.liked_songs):
            self.liked_songs.append(song)

    def unlike_song(self, song_id: str) -> None:
        self.liked_songs = [
            song for song in self.liked_songs if song.song_id != song_id
        ]

    def save_album(self, album: Album) -> None:
        if all(existing.album_id != album.album_id for existing in self.saved_albums):
            self.saved_albums.append(album)

    def follow_artist(self, artist: Artist) -> None:
        if all(
            existing.artist_id != artist.artist_id
            for existing in self.followed_artists
        ):
            self.followed_artists.append(artist)


@dataclass
class PlayHistoryItem:
    song: Song
    played_at: datetime
    device_id: Optional[str] = None


@dataclass
class PlayHistory:
    items: List[PlayHistoryItem] = field(default_factory=list)

    def add_play(self, song: Song, device_id: Optional[str] = None) -> None:
        self.items.append(
            PlayHistoryItem(song=song, played_at=datetime.now(), device_id=device_id)
        )

    def get_recent(self, limit: int = 10) -> List[PlayHistoryItem]:
        return self.items[-limit:][::-1]


@dataclass
class DownloadedSong:
    song: Song
    downloaded_at: datetime


@dataclass
class DownloadLibrary:
    songs: Dict[str, DownloadedSong] = field(default_factory=dict)

    def add_song(self, song: Song) -> None:
        self.songs[song.song_id] = DownloadedSong(
            song=song,
            downloaded_at=datetime.now(),
        )

    def has_song(self, song_id: str) -> bool:
        return song_id in self.songs

    def remove_song(self, song_id: str) -> None:
        self.songs.pop(song_id, None)


# =========================
# Queue / Playback
# =========================

@dataclass
class PlayQueue:
    songs: List[Song] = field(default_factory=list)
    current_index: int = -1
    shuffle_enabled: bool = False
    repeat_mode: RepeatMode = RepeatMode.OFF

    def load(
        self,
        songs: List[Song],
        start_index: int = 0,
        shuffle_enabled: bool = False,
    ) -> None:
        self.songs = songs[:]
        self.shuffle_enabled = shuffle_enabled

        if shuffle_enabled and self.songs:
            current_song = (
                self.songs[start_index]
                if 0 <= start_index < len(self.songs)
                else self.songs[0]
            )
            remaining = [
                song for i, song in enumerate(self.songs) if i != start_index
            ]
            random.shuffle(remaining)
            self.songs = [current_song] + remaining
            self.current_index = 0
        else:
            self.current_index = (
                start_index if songs and 0 <= start_index < len(songs) else -1
            )

    def add_to_queue(self, song: Song) -> None:
        self.songs.append(song)
        if self.current_index == -1:
            self.current_index = 0

    def get_current_song(self) -> Optional[Song]:
        if 0 <= self.current_index < len(self.songs):
            return self.songs[self.current_index]
        return None

    def next_song(self) -> Optional[Song]:
        if not self.songs:
            return None

        if self.repeat_mode == RepeatMode.ONE:
            return self.get_current_song()

        if self.current_index + 1 < len(self.songs):
            self.current_index += 1
            return self.songs[self.current_index]

        if self.repeat_mode == RepeatMode.ALL:
            self.current_index = 0
            return self.songs[self.current_index]

        return None

    def previous_song(self) -> Optional[Song]:
        if not self.songs:
            return None

        if self.current_index - 1 >= 0:
            self.current_index -= 1
            return self.songs[self.current_index]

        if self.repeat_mode == RepeatMode.ALL and self.songs:
            self.current_index = len(self.songs) - 1
            return self.songs[self.current_index]

        return None

    def clear(self) -> None:
        self.songs.clear()
        self.current_index = -1


@dataclass
class PlaybackSession:
    user_id: str
    device: Device
    queue: PlayQueue = field(default_factory=PlayQueue)
    state: PlaybackState = PlaybackState.STOPPED
    current_position_seconds: int = 0

    def play(self) -> None:
        if self.queue.get_current_song() is not None:
            self.state = PlaybackState.PLAYING

    def pause(self) -> None:
        if self.state == PlaybackState.PLAYING:
            self.state = PlaybackState.PAUSED

    def stop(self) -> None:
        self.state = PlaybackState.STOPPED
        self.current_position_seconds = 0

    def resume(self) -> None:
        if self.state == PlaybackState.PAUSED:
            self.state = PlaybackState.PLAYING


# =========================
# User
# =========================

@dataclass
class User:
    user_id: str
    username: str
    subscription: Subscription
    library: Library = field(default_factory=Library)
    playlists: List[Playlist] = field(default_factory=list)
    play_history: PlayHistory = field(default_factory=PlayHistory)
    downloads: DownloadLibrary = field(default_factory=DownloadLibrary)
    devices: List[Device] = field(default_factory=list)

    def add_device(self, device: Device) -> None:
        if all(existing.device_id != device.device_id for existing in self.devices):
            self.devices.append(device)

    def add_playlist_reference(self, playlist: Playlist) -> None:
        if all(existing.playlist_id != playlist.playlist_id for existing in self.playlists):
            self.playlists.append(playlist)

    def create_playlist(
        self,
        playlist_id: str,
        name: str,
        is_public: bool = False,
    ) -> Playlist:
        playlist = Playlist(
            playlist_id=playlist_id,
            name=name,
            owner_user_id=self.user_id,
            is_public=is_public,
        )
        self.playlists.append(playlist)
        return playlist

    def get_playlist(self, playlist_id: str) -> Optional[Playlist]:
        for playlist in self.playlists:
            if playlist.playlist_id == playlist_id:
                return playlist
        return None


# =========================
# Catalog
# =========================

class Catalog:
    def __init__(self) -> None:
        self.songs: Dict[str, Song] = {}
        self.albums: Dict[str, Album] = {}
        self.artists: Dict[str, Artist] = {}

    def add_artist(self, artist: Artist) -> None:
        self.artists[artist.artist_id] = artist

    def add_song(self, song: Song) -> None:
        self.songs[song.song_id] = song

    def add_album(self, album: Album) -> None:
        self.albums[album.album_id] = album

    def get_song(self, song_id: str) -> Optional[Song]:
        return self.songs.get(song_id)

    def get_album(self, album_id: str) -> Optional[Album]:
        return self.albums.get(album_id)

    def get_artist(self, artist_id: str) -> Optional[Artist]:
        return self.artists.get(artist_id)

    def search_songs(self, query: str) -> List[Song]:
        q = query.lower()
        return [song for song in self.songs.values() if q in song.title.lower()]

    def search_artists(self, query: str) -> List[Artist]:
        q = query.lower()
        return [artist for artist in self.artists.values() if q in artist.name.lower()]

    def search_albums(self, query: str) -> List[Album]:
        q = query.lower()
        return [album for album in self.albums.values() if q in album.title.lower()]


# =========================
# Playback Policy Strategy
# =========================

class PlaybackPolicy(ABC):
    @abstractmethod
    def validate_play(
        self,
        user: User,
        active_sessions: List[PlaybackSession],
    ) -> None:
        pass

    @abstractmethod
    def can_download(self) -> bool:
        pass

    @abstractmethod
    def max_concurrent_streams(self) -> int:
        pass


class FreePlaybackPolicy(PlaybackPolicy):
    def validate_play(
        self,
        user: User,
        active_sessions: List[PlaybackSession],
    ) -> None:
        if not user.subscription.is_active():
            raise ValueError("Subscription is not active")
        if len(active_sessions) >= 1:
            raise ValueError("Free plan supports only one active stream")

    def can_download(self) -> bool:
        return False

    def max_concurrent_streams(self) -> int:
        return 1


class PremiumPlaybackPolicy(PlaybackPolicy):
    def validate_play(
        self,
        user: User,
        active_sessions: List[PlaybackSession],
    ) -> None:
        if not user.subscription.is_active():
            raise ValueError("Subscription is not active")
        if len(active_sessions) >= 3:
            raise ValueError("Premium concurrent stream limit reached")

    def can_download(self) -> bool:
        return True

    def max_concurrent_streams(self) -> int:
        return 3


# =========================
# Service
# =========================

class MusicStreamingService:
    def __init__(self, catalog: Catalog) -> None:
        self.catalog = catalog
        self.users: Dict[str, User] = {}
        self.sessions_by_user: Dict[str, Dict[str, PlaybackSession]] = {}
        self.playlists: Dict[str, Playlist] = {}

    def add_user(self, user: User) -> None:
        self.users[user.user_id] = user
        self.sessions_by_user[user.user_id] = {}
        for playlist in user.playlists:
            self.playlists[playlist.playlist_id] = playlist

    def get_user(self, user_id: str) -> Optional[User]:
        return self.users.get(user_id)

    def create_playlist(
        self,
        owner_user_id: str,
        playlist_id: str,
        name: str,
        is_public: bool = False,
    ) -> Playlist:
        if playlist_id in self.playlists:
            raise ValueError("Playlist ID already exists")

        user = self.users[owner_user_id]
        playlist = user.create_playlist(playlist_id, name, is_public)
        self.playlists[playlist_id] = playlist
        return playlist

    def get_playlist(self, playlist_id: str) -> Optional[Playlist]:
        return self.playlists.get(playlist_id)

    def _get_policy(self, user: User) -> PlaybackPolicy:
        if user.subscription.subscription_type == SubscriptionType.PREMIUM:
            return PremiumPlaybackPolicy()
        return FreePlaybackPolicy()

    def _get_user_sessions(self, user_id: str) -> Dict[str, PlaybackSession]:
        if user_id not in self.sessions_by_user:
            raise ValueError("User not found")
        return self.sessions_by_user[user_id]

    def _get_or_create_session(
        self,
        user: User,
        device: Device,
    ) -> PlaybackSession:
        sessions = self._get_user_sessions(user.user_id)
        if device.device_id not in sessions:
            sessions[device.device_id] = PlaybackSession(
                user_id=user.user_id,
                device=device,
            )
        return sessions[device.device_id]

    def _get_active_sessions(self, user_id: str) -> List[PlaybackSession]:
        sessions = self._get_user_sessions(user_id)
        return [
            session
            for session in sessions.values()
            if session.state in {PlaybackState.PLAYING, PlaybackState.PAUSED}
        ]

    def register_device(self, user_id: str, device: Device) -> None:
        user = self.users[user_id]
        user.add_device(device)

    def play_song(self, user_id: str, device: Device, song_id: str) -> None:
        user = self.users[user_id]
        policy = self._get_policy(user)
        active_sessions = self._get_active_sessions(user_id)
        policy.validate_play(
            user,
            [
                session
                for session in active_sessions
                if session.device.device_id != device.device_id
            ],
        )

        song = self.catalog.get_song(song_id)
        if song is None:
            raise ValueError("Song not found")

        session = self._get_or_create_session(user, device)
        session.queue.load([song], 0)
        session.current_position_seconds = 0
        session.play()
        user.play_history.add_play(song, device.device_id)

    def play_album(
        self,
        user_id: str,
        device: Device,
        album_id: str,
        start_index: int = 0,
        shuffle_enabled: bool = False,
    ) -> None:
        user = self.users[user_id]
        policy = self._get_policy(user)
        active_sessions = self._get_active_sessions(user_id)
        policy.validate_play(
            user,
            [
                session
                for session in active_sessions
                if session.device.device_id != device.device_id
            ],
        )

        album = self.catalog.get_album(album_id)
        if album is None:
            raise ValueError("Album not found")
        if not album.songs:
            raise ValueError("Album has no songs")

        session = self._get_or_create_session(user, device)
        session.queue.load(
            album.songs,
            start_index,
            shuffle_enabled=shuffle_enabled,
        )
        session.current_position_seconds = 0
        session.play()

        current_song = session.queue.get_current_song()
        if current_song:
            user.play_history.add_play(current_song, device.device_id)

    def play_playlist(
        self,
        user_id: str,
        device: Device,
        playlist_id: str,
        start_index: int = 0,
        shuffle_enabled: bool = False,
    ) -> None:
        user = self.users[user_id]
        policy = self._get_policy(user)
        active_sessions = self._get_active_sessions(user_id)
        policy.validate_play(
            user,
            [
                session
                for session in active_sessions
                if session.device.device_id != device.device_id
            ],
        )

        playlist = self.get_playlist(playlist_id)
        if playlist is None:
            raise ValueError("Playlist not found")
        if not playlist.can_view(user_id):
            raise PermissionError("User cannot access playlist")
        if not playlist.songs:
            raise ValueError("Playlist is empty")

        session = self._get_or_create_session(user, device)
        session.queue.load(
            playlist.songs,
            start_index,
            shuffle_enabled=shuffle_enabled,
        )
        session.current_position_seconds = 0
        session.play()

        current_song = session.queue.get_current_song()
        if current_song:
            user.play_history.add_play(current_song, device.device_id)

    def pause(self, user_id: str, device_id: str) -> None:
        self._get_user_sessions(user_id)[device_id].pause()

    def resume(self, user_id: str, device_id: str) -> None:
        self._get_user_sessions(user_id)[device_id].resume()

    def stop(self, user_id: str, device_id: str) -> None:
        self._get_user_sessions(user_id)[device_id].stop()

    def next_song(self, user_id: str, device_id: str) -> Optional[Song]:
        user = self.users[user_id]
        session = self._get_user_sessions(user_id)[device_id]
        next_song = session.queue.next_song()
        session.current_position_seconds = 0

        if next_song:
            session.play()
            user.play_history.add_play(next_song, device_id)
        else:
            session.stop()

        return next_song

    def previous_song(self, user_id: str, device_id: str) -> Optional[Song]:
        session = self._get_user_sessions(user_id)[device_id]
        song = session.queue.previous_song()
        session.current_position_seconds = 0
        if song:
            session.play()
        return song

    def set_repeat_mode(
        self,
        user_id: str,
        device_id: str,
        repeat_mode: RepeatMode,
    ) -> None:
        self._get_user_sessions(user_id)[device_id].queue.repeat_mode = repeat_mode

    def add_song_to_queue(self, user_id: str, device_id: str, song_id: str) -> None:
        song = self.catalog.get_song(song_id)
        if song is None:
            raise ValueError("Song not found")
        self._get_user_sessions(user_id)[device_id].queue.add_to_queue(song)

    def add_song_to_playlist(
        self,
        acting_user_id: str,
        playlist_id: str,
        song_id: str,
    ) -> None:
        playlist = self.get_playlist(playlist_id)
        if playlist is None:
            raise ValueError("Playlist not found")

        song = self.catalog.get_song(song_id)
        if song is None:
            raise ValueError("Song not found")

        playlist.add_song(acting_user_id, song)

    def remove_song_from_playlist(
        self,
        acting_user_id: str,
        playlist_id: str,
        song_id: str,
    ) -> None:
        playlist = self.get_playlist(playlist_id)
        if playlist is None:
            raise ValueError("Playlist not found")
        playlist.remove_song(acting_user_id, song_id)

    def add_collaborator(
        self,
        owner_user_id: str,
        playlist_id: str,
        collaborator_user_id: str,
    ) -> None:
        if collaborator_user_id not in self.users:
            raise ValueError("Collaborator user not found")

        playlist = self.get_playlist(playlist_id)
        if playlist is None:
            raise ValueError("Playlist not found")
        if playlist.owner_user_id != owner_user_id:
            raise PermissionError("Only owner can add collaborators")

        playlist.add_collaborator(
            collaborator_user_id,
            PlaylistRole.COLLABORATOR,
        )
        self.users[collaborator_user_id].add_playlist_reference(playlist)

    def like_song(self, user_id: str, song_id: str) -> None:
        user = self.users[user_id]
        song = self.catalog.get_song(song_id)
        if song is None:
            raise ValueError("Song not found")
        user.library.like_song(song)

    def unlike_song(self, user_id: str, song_id: str) -> None:
        self.users[user_id].library.unlike_song(song_id)

    def save_album(self, user_id: str, album_id: str) -> None:
        user = self.users[user_id]
        album = self.catalog.get_album(album_id)
        if album is None:
            raise ValueError("Album not found")
        user.library.save_album(album)

    def follow_artist(self, user_id: str, artist_id: str) -> None:
        user = self.users[user_id]
        artist = self.catalog.get_artist(artist_id)
        if artist is None:
            raise ValueError("Artist not found")
        user.library.follow_artist(artist)

    def download_song(self, user_id: str, song_id: str) -> None:
        user = self.users[user_id]
        policy = self._get_policy(user)
        if not policy.can_download():
            raise ValueError("Downloads are available only for premium users")

        song = self.catalog.get_song(song_id)
        if song is None:
            raise ValueError("Song not found")

        user.downloads.add_song(song)

    def play_downloaded_song(self, user_id: str, device: Device, song_id: str) -> None:
        user = self.users[user_id]
        if not user.downloads.has_song(song_id):
            raise ValueError("Song is not downloaded")

        song = self.catalog.get_song(song_id)
        if song is None:
            raise ValueError("Song metadata not found")

        session = self._get_or_create_session(user, device)
        session.queue.load([song], 0)
        session.current_position_seconds = 0
        session.play()
        user.play_history.add_play(song, device.device_id)

    def search(self, query: str) -> Dict[str, List]:
        return {
            "songs": self.catalog.search_songs(query),
            "artists": self.catalog.search_artists(query),
            "albums": self.catalog.search_albums(query),
        }


# =========================
# Example Usage
# =========================

if __name__ == "__main__":
    artist = Artist(artist_id="a1", name="The Weeknd")

    song1 = Song(
        song_id="s1",
        title="Blinding Lights",
        duration_seconds=200,
        artist=artist,
        album_id="al1",
    )
    song2 = Song(
        song_id="s2",
        title="Save Your Tears",
        duration_seconds=215,
        artist=artist,
        album_id="al1",
    )
    song3 = Song(
        song_id="s3",
        title="In Your Eyes",
        duration_seconds=220,
        artist=artist,
        album_id="al1",
    )

    album = Album(album_id="al1", title="After Hours", artist=artist)
    album.add_song(song1)
    album.add_song(song2)
    album.add_song(song3)

    catalog = Catalog()
    catalog.add_artist(artist)
    catalog.add_song(song1)
    catalog.add_song(song2)
    catalog.add_song(song3)
    catalog.add_album(album)

    subscription = Subscription(
        subscription_type=SubscriptionType.PREMIUM,
        status=SubscriptionStatus.ACTIVE,
        start_date=datetime.now(),
        end_date=datetime.now() + timedelta(days=30),
    )

    owner = User(user_id="u1", username="vaibhav", subscription=subscription)
    collaborator = User(user_id="u2", username="alex", subscription=subscription)

    device1 = Device(
        device_id="d1",
        name="iPhone",
        device_type=DeviceType.MOBILE,
    )
    device2 = Device(
        device_id="d2",
        name="MacBook",
        device_type=DeviceType.DESKTOP,
    )
    owner.add_device(device1)
    collaborator.add_device(device2)

    service = MusicStreamingService(catalog)
    service.add_user(owner)
    service.add_user(collaborator)

    playlist = service.create_playlist("u1", "p1", "Favorites", is_public=True)
    service.add_song_to_playlist("u1", "p1", "s1")
    service.add_song_to_playlist("u1", "p1", "s2")

    service.add_collaborator("u1", "p1", "u2")
    service.add_song_to_playlist("u2", "p1", "s3")

    service.play_song("u1", device1, "s1")
    print(
        "Current song on owner device:",
        service._get_user_sessions("u1")["d1"].queue.get_current_song(),
    )

    service.play_playlist("u2", device2, "p1", shuffle_enabled=True)
    print(
        "Current song on collaborator device:",
        service._get_user_sessions("u2")["d2"].queue.get_current_song(),
    )

    print("Playlist songs:", [song.title for song in service.get_playlist("p1").songs])
```

## Design Principles, Design Patterns, SOLID Principles, and OOP Concepts Used in the Music Streaming Service LLD

# 1. Design Principles Used

## 1.1 Separation of Concerns

Different responsibilities are split into different classes.

Examples:
- `Catalog` manages songs, albums, and artists
- `MusicStreamingService` coordinates the overall workflow
- `PlayQueue` manages queue navigation
- `PlaybackSession` manages session state
- `Library` manages liked songs, albums, and followed artists
- `DownloadLibrary` manages offline downloads
- `Playlist` manages playlist specific behavior
- `PlaybackPolicy` handles subscription based playback rules

**Why this is good**  
Each class has a focused job, so the system is easier to understand, extend, and test.

---

## 1.2 Composition Over Inheritance

The design mainly builds complex behavior by combining smaller objects.

Examples:
- `User` contains `Library`, `PlayHistory`, `DownloadLibrary`
- `PlaybackSession` contains `PlayQueue`
- `Album` contains songs
- `Playlist` contains songs and collaborators
- `MusicStreamingService` contains catalog, users, sessions, playlists
- `Song` contains `Artist`

**Why this is good**  
Composition keeps the model flexible and avoids deep inheritance chains.

---

## 1.3 Encapsulation of Business Rules

Important business rules are kept close to the classes that own them.

Examples:
- `Subscription.is_active()` handles subscription validity
- `Playlist.can_edit()` and `Playlist.can_view()` handle access control
- `PlayQueue.next_song()` handles repeat mode rules
- `PlaybackPolicy` handles concurrent stream and download rules

**Why this is good**  
Rules stay centralized instead of being scattered across the codebase.

---

## 1.4 High Cohesion

Classes are designed so that their methods are strongly related to their internal data.

Examples:
- `PlayHistory` only deals with play tracking
- `DownloadLibrary` only deals with downloaded songs
- `Catalog` only deals with catalog search and retrieval
- `PlaybackSession` only deals with playback state

**Why this is good**  
Highly cohesive classes are easier to maintain and reason about.

---

## 1.5 Low Coupling

The classes are designed to reduce unnecessary dependencies.

Examples:
- playback rules are abstracted behind `PlaybackPolicy`
- service logic uses `Catalog` instead of directly managing songs internally
- playlists are looked up through a registry instead of hardcoded ownership assumptions

**Why this is good**  
Low coupling makes the design easier to modify without breaking many parts.

---

# 2. Design Patterns Used

## 2.1 Strategy Pattern

**Where used**
- `PlaybackPolicy`
- `FreePlaybackPolicy`
- `PremiumPlaybackPolicy`

**Why it fits**
Playback restrictions differ by subscription type.

Examples:
- free user may have only one active stream
- premium user may have more active streams
- free user cannot download songs
- premium user can download songs

Instead of hardcoding all such rules in `MusicStreamingService`, the system delegates them to policy classes.

**Benefits**
- easy to add more policies later
- keeps service logic cleaner
- supports future plans like family, student, business

---

## 2.2 Facade Pattern

**Where used**
- `MusicStreamingService`

**Why it fits**
It provides a single entry point for many operations:
- play song
- play album
- play playlist
- add to queue
- like song
- save album
- follow artist
- download song
- search
- manage collaborators

Clients do not need to directly coordinate many lower level classes.

**Benefits**
- simpler API
- cleaner workflow
- hides internal complexity

---

## 2.3 Repository-like Pattern

**Where used**
- `Catalog`
- playlist registry inside `MusicStreamingService`

**Why it fits**
These classes act as in-memory stores for domain objects.

Examples:
- `Catalog` stores songs, albums, artists
- `MusicStreamingService.playlists` acts as a registry for playlists

This is not a full repository implementation, but it follows the same idea of central object retrieval and management.

---

## 2.4 Aggregate / Ownership Pattern

**Where used**
- `User`
- `Album`
- `Playlist`

**Why it fits**
These objects own and manage related sub-objects.

Examples:
- `User` owns personal library, history, downloads, devices
- `Album` owns songs
- `Playlist` owns playlist song collection and collaborator rules

This helps model domain boundaries clearly.

---

## 2.5 State-like Behavior

**Where used**
- `PlaybackSession.state`
- `PlaybackState`

**Why it fits**
Playback behavior depends on current state:
- playing
- paused
- stopped

This is not a full State pattern with separate state classes, but it is state-driven design.

---

# 3. SOLID Principles Used

## 3.1 Single Responsibility Principle (SRP)

Each class mostly has one reason to change.

Examples:
- `Subscription` only handles subscription status
- `Artist` only models artist data
- `Song` only models song data
- `Album` only models album data
- `Playlist` only handles playlist state and permissions
- `Library` only handles user library
- `PlayHistory` only handles history
- `DownloadLibrary` only handles downloads
- `PlayQueue` only handles queue behavior
- `PlaybackSession` only handles playback state
- `Catalog` only handles catalog management
- `MusicStreamingService` coordinates use cases

**Why this is good**  
Each class stays focused and easier to maintain.

---

## 3.2 Open Closed Principle (OCP)

The design is open for extension without requiring major modification of stable code.

Examples:
- new playback policies can be added by extending `PlaybackPolicy`
- new subscription types can map to new policies
- new device types can be added in `DeviceType`
- new repeat modes can be added in `RepeatMode`
- new playlist access roles can be added in `PlaylistRole`

**Why this is good**  
The system can evolve with less risk.

---

## 3.3 Liskov Substitution Principle (LSP)

Subclasses or implementations should work correctly in place of their base abstractions.

Examples:
- `FreePlaybackPolicy` and `PremiumPlaybackPolicy` can both be used wherever `PlaybackPolicy` is expected

**Why this is good**  
The service logic can rely on abstractions without worrying about the specific concrete implementation.

---

## 3.4 Interface Segregation Principle (ISP)

Interfaces should remain small and focused.

Example:
- `PlaybackPolicy` has only the methods needed for playback rules:
  - `validate_play()`
  - `can_download()`
  - `max_concurrent_streams()`

**Why this is good**  
The abstraction stays clean and easy to implement.

---

## 3.5 Dependency Inversion Principle (DIP)

Higher level logic should depend on abstractions where possible.

Examples:
- `MusicStreamingService` depends on `PlaybackPolicy` abstraction rather than embedding all playback rule details directly
- the policy selection is separated from the streaming workflow

**Where it can improve**
The service still directly owns concrete registries like `Catalog` and session maps. In a larger system, these could be abstracted further behind repositories or service interfaces.

---

# 4. OOP Concepts Used

## 4.1 Encapsulation

Each class bundles its data and behavior together.

Examples:
- `Playlist` holds songs and collaborator permissions together
- `PlayQueue` holds queue data and queue navigation logic together
- `PlaybackSession` holds session state and playback actions together
- `Library` holds liked songs and related management methods together

**Why this is good**  
It protects internal consistency and keeps logic organized.

---

## 4.2 Abstraction

The design uses abstractions to hide implementation details.

Examples:
- `PlaybackPolicy` is an abstraction for playback rule behavior

**Why this is good**  
It allows swapping implementations without changing calling code.

---

## 4.3 Inheritance

Inheritance is used in a limited and controlled way.

Examples:
- `FreePlaybackPolicy` inherits from `PlaybackPolicy`
- `PremiumPlaybackPolicy` inherits from `PlaybackPolicy`

**Why this is good**  
The design prefers composition first, using inheritance only where it adds clear value.

---

## 4.4 Polymorphism

The same interface can behave differently depending on implementation.

Examples:
- `validate_play()` behaves differently for free vs premium users
- `can_download()` behaves differently for different policies

**Why this is good**  
This keeps rules extensible and clean.

---

## 4.5 Composition

Composition is heavily used across the design.

Examples:
- `User` contains library, playlists, downloads, history, devices
- `PlaybackSession` contains queue
- `Album` contains songs
- `Playlist` contains songs and collaborators
- `MusicStreamingService` contains registries and sessions

**Why this is good**  
This is one of the strongest OOP aspects of the design.

---

# 5. Additional Strong Design Choices

## 5.1 Global Playlist Registry

**Where used**
- `MusicStreamingService.playlists`

**Why it is useful**
This supports collaborative playlists correctly across users.

Without a global registry, playlist access would be limited to the owner’s local list only.

---

## 5.2 Session Per Device

**Where used**
- `sessions_by_user[user_id][device_id]`

**Why it is useful**
It models real world music streaming better and supports:
- device switching
- concurrent playback rules
- tracking history by device

---

## 5.3 Policy Based Subscription Rules

**Where used**
- `PlaybackPolicy`

**Why it is useful**
It cleanly separates entitlement logic from main playback flow.

---

## 5.4 Queue Features Embedded in Queue

**Where used**
- `PlayQueue`

**Why it is useful**
Shuffle and repeat logic belong naturally in queue behavior instead of cluttering the service class.

---

# 6. Interview Ready Summary

You can say:

> This music streaming LLD follows strong separation of concerns and composition based design. The main design patterns are Strategy through `PlaybackPolicy` for free vs premium playback rules, Facade through `MusicStreamingService` as the central API, and repository-like registries for catalog and playlists. It follows SOLID principles well, especially SRP, OCP, and partial DIP. In OOP terms, it uses encapsulation, abstraction, limited inheritance, polymorphism, and strong composition to keep the system modular and extensible.