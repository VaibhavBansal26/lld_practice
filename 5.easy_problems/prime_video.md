# 13. Prime Video

Code

```python
from __future__ import annotations

from abc import ABC
from dataclasses import dataclass, field
from datetime import datetime, timedelta
from enum import Enum
from typing import Dict, List, Optional
import uuid


# =========================
# Enums
# =========================

class SubscriptionStatus(str, Enum):
    ACTIVE = "ACTIVE"
    EXPIRED = "EXPIRED"
    CANCELLED = "CANCELLED"


class ContentType(str, Enum):
    MOVIE = "MOVIE"
    SERIES = "SERIES"
    EPISODE = "EPISODE"


class DeviceType(str, Enum):
    MOBILE = "MOBILE"
    TABLET = "TABLET"
    WEB = "WEB"
    TV = "TV"


class VideoQuality(str, Enum):
    SD = "SD"
    HD = "HD"
    UHD = "UHD"


class SessionStatus(str, Enum):
    ACTIVE = "ACTIVE"
    STOPPED = "STOPPED"
    COMPLETED = "COMPLETED"


# =========================
# Core User / Subscription / Device
# =========================

@dataclass
class Subscription:
    subscription_id: str
    status: SubscriptionStatus
    start_date: datetime
    end_date: datetime

    def is_active(self) -> bool:
        now = datetime.now()
        return self.status == SubscriptionStatus.ACTIVE and self.start_date <= now <= self.end_date

    def expire_if_needed(self) -> None:
        if datetime.now() > self.end_date and self.status == SubscriptionStatus.ACTIVE:
            self.status = SubscriptionStatus.EXPIRED


@dataclass(frozen=True)
class Device:
    device_id: str
    name: str
    device_type: DeviceType


@dataclass
class User:
    user_id: str
    name: str
    email: str
    subscription: Subscription
    devices: List[Device] = field(default_factory=list)

    def add_device(self, device: Device) -> None:
        self.devices.append(device)

    def get_device(self, device_id: str) -> Optional[Device]:
        for device in self.devices:
            if device.device_id == device_id:
                return device
        return None


# =========================
# Content Hierarchy
# =========================

@dataclass
class Content(ABC):
    content_id: str
    title: str
    description: str
    genre: str
    release_year: int
    rating: float
    duration_minutes: int
    content_type: ContentType


@dataclass
class Movie(Content):
    pass


@dataclass
class Episode(Content):
    episode_number: int = 0


@dataclass
class Season:
    season_number: int
    episodes: List[Episode] = field(default_factory=list)

    def add_episode(self, episode: Episode) -> None:
        self.episodes.append(episode)

    def get_episode(self, episode_number: int) -> Optional[Episode]:
        for episode in self.episodes:
            if episode.episode_number == episode_number:
                return episode
        return None


@dataclass
class Series(Content):
    seasons: List[Season] = field(default_factory=list)

    def add_season(self, season: Season) -> None:
        self.seasons.append(season)

    def get_season(self, season_number: int) -> Optional[Season]:
        for season in self.seasons:
            if season.season_number == season_number:
                return season
        return None


# =========================
# Watchlist / Watch History
# =========================

@dataclass
class WatchlistItem:
    content: Content
    added_at: datetime = field(default_factory=datetime.now)


@dataclass
class WatchProgress:
    content_id: str
    progress_seconds: int
    completed: bool = False
    updated_at: datetime = field(default_factory=datetime.now)


@dataclass
class WatchHistory:
    user_id: str
    progress_map: Dict[str, WatchProgress] = field(default_factory=dict)

    def update_progress(
        self,
        content_id: str,
        progress_seconds: int,
        completed: bool = False,
    ) -> None:
        self.progress_map[content_id] = WatchProgress(
            content_id=content_id,
            progress_seconds=progress_seconds,
            completed=completed,
            updated_at=datetime.now(),
        )

    def get_progress(self, content_id: str) -> Optional[WatchProgress]:
        return self.progress_map.get(content_id)

    def get_all_progress(self) -> List[WatchProgress]:
        return list(self.progress_map.values())


# =========================
# Playback
# =========================

@dataclass
class PlaybackSession:
    session_id: str
    user: User
    content: Content
    device: Device
    started_at: datetime
    quality: VideoQuality
    current_position_seconds: int = 0
    status: SessionStatus = SessionStatus.ACTIVE

    def stop(self) -> None:
        self.status = SessionStatus.STOPPED

    def complete(self) -> None:
        self.status = SessionStatus.COMPLETED

    def is_active(self) -> bool:
        return self.status == SessionStatus.ACTIVE


# =========================
# Catalog
# =========================

class Catalog:
    def __init__(self):
        self.contents: Dict[str, Content] = {}

    def add_content(self, content: Content) -> None:
        self.contents[content.content_id] = content

    def get_content_by_id(self, content_id: str) -> Optional[Content]:
        return self.contents.get(content_id)

    def get_all_content(self) -> List[Content]:
        return list(self.contents.values())

    def search_by_title(self, query: str) -> List[Content]:
        query = query.lower()
        return [content for content in self.contents.values() if query in content.title.lower()]

    def filter_by_genre(self, genre: str) -> List[Content]:
        return [
            content for content in self.contents.values()
            if content.genre.lower() == genre.lower()
        ]

    def filter_by_type(self, content_type: ContentType) -> List[Content]:
        return [
            content for content in self.contents.values()
            if content.content_type == content_type
        ]


# =========================
# Repositories
# =========================

class WatchlistRepository:
    def __init__(self):
        self.watchlists: Dict[str, List[WatchlistItem]] = {}

    def add_item(self, user_id: str, item: WatchlistItem) -> None:
        if user_id not in self.watchlists:
            self.watchlists[user_id] = []
        self.watchlists[user_id].append(item)

    def remove_item(self, user_id: str, content_id: str) -> None:
        if user_id not in self.watchlists:
            return
        self.watchlists[user_id] = [
            item for item in self.watchlists[user_id]
            if item.content.content_id != content_id
        ]

    def get_watchlist(self, user_id: str) -> List[WatchlistItem]:
        return list(self.watchlists.get(user_id, []))

    def exists(self, user_id: str, content_id: str) -> bool:
        return any(
            item.content.content_id == content_id
            for item in self.watchlists.get(user_id, [])
        )


class WatchHistoryRepository:
    def __init__(self):
        self.histories: Dict[str, WatchHistory] = {}

    def get_or_create(self, user_id: str) -> WatchHistory:
        if user_id not in self.histories:
            self.histories[user_id] = WatchHistory(user_id=user_id)
        return self.histories[user_id]


class PlaybackSessionRepository:
    def __init__(self):
        self.sessions: Dict[str, PlaybackSession] = {}

    def save(self, session: PlaybackSession) -> None:
        self.sessions[session.session_id] = session

    def get_by_id(self, session_id: str) -> Optional[PlaybackSession]:
        return self.sessions.get(session_id)

    def get_active_sessions_for_user(self, user_id: str) -> List[PlaybackSession]:
        return [
            session
            for session in self.sessions.values()
            if session.user.user_id == user_id and session.is_active()
        ]


# =========================
# Service
# =========================

class PrimeVideoService:
    def __init__(self, catalog: Catalog):
        self.catalog = catalog
        self.watchlist_repository = WatchlistRepository()
        self.watch_history_repository = WatchHistoryRepository()
        self.playback_session_repository = PlaybackSessionRepository()

    # ---------- Content / Browse ----------
    def search(self, query: str) -> List[Content]:
        return self.catalog.search_by_title(query)

    def browse_by_genre(self, genre: str) -> List[Content]:
        return self.catalog.filter_by_genre(genre)

    def browse_movies(self) -> List[Content]:
        return self.catalog.filter_by_type(ContentType.MOVIE)

    def browse_series(self) -> List[Content]:
        return self.catalog.filter_by_type(ContentType.SERIES)

    def get_content_details(self, content_id: str) -> Optional[Content]:
        return self.catalog.get_content_by_id(content_id)

    # ---------- Watchlist ----------
    def add_to_watchlist(self, user: User, content_id: str) -> None:
        content = self.catalog.get_content_by_id(content_id)
        if content is None:
            raise ValueError("Content not found")

        if self.watchlist_repository.exists(user.user_id, content_id):
            return

        self.watchlist_repository.add_item(
            user.user_id,
            WatchlistItem(content=content),
        )

    def remove_from_watchlist(self, user: User, content_id: str) -> None:
        self.watchlist_repository.remove_item(user.user_id, content_id)

    def get_watchlist(self, user: User) -> List[WatchlistItem]:
        return self.watchlist_repository.get_watchlist(user.user_id)

    # ---------- Playback ----------
    def start_playback(
        self,
        user: User,
        content_id: str,
        device_id: str,
        quality: VideoQuality,
    ) -> PlaybackSession:
        user.subscription.expire_if_needed()
        if not user.subscription.is_active():
            raise ValueError("User subscription is not active")

        content = self.catalog.get_content_by_id(content_id)
        if content is None:
            raise ValueError("Content not found")

        device = user.get_device(device_id)
        if device is None:
            raise ValueError("Device not found")

        history = self.watch_history_repository.get_or_create(user.user_id)
        progress = history.get_progress(content_id)
        start_position = progress.progress_seconds if progress else 0

        session = PlaybackSession(
            session_id=str(uuid.uuid4()),
            user=user,
            content=content,
            device=device,
            started_at=datetime.now(),
            quality=quality,
            current_position_seconds=start_position,
            status=SessionStatus.ACTIVE,
        )
        self.playback_session_repository.save(session)
        return session

    def update_playback_progress(
        self,
        session_id: str,
        progress_seconds: int,
        completed: bool = False,
    ) -> None:
        session = self.playback_session_repository.get_by_id(session_id)
        if session is None or not session.is_active():
            raise ValueError("Invalid or inactive session")

        session.current_position_seconds = progress_seconds
        history = self.watch_history_repository.get_or_create(session.user.user_id)
        history.update_progress(
            content_id=session.content.content_id,
            progress_seconds=progress_seconds,
            completed=completed,
        )

        if completed:
            session.complete()

        self.playback_session_repository.save(session)

    def stop_playback(self, session_id: str) -> None:
        session = self.playback_session_repository.get_by_id(session_id)
        if session is None:
            raise ValueError("Session not found")

        history = self.watch_history_repository.get_or_create(session.user.user_id)
        history.update_progress(
            content_id=session.content.content_id,
            progress_seconds=session.current_position_seconds,
            completed=False,
        )
        session.stop()
        self.playback_session_repository.save(session)

    def get_watch_history(self, user: User) -> WatchHistory:
        return self.watch_history_repository.get_or_create(user.user_id)

    def get_continue_watching(self, user: User) -> List[WatchProgress]:
        history = self.watch_history_repository.get_or_create(user.user_id)
        return [progress for progress in history.get_all_progress() if not progress.completed]


# =========================
# Demo
# =========================

if __name__ == "__main__":
    # subscription
    subscription = Subscription(
        subscription_id="sub1",
        status=SubscriptionStatus.ACTIVE,
        start_date=datetime.now() - timedelta(days=10),
        end_date=datetime.now() + timedelta(days=20),
    )

    # user
    user = User(
        user_id="u1",
        name="Vaibhav",
        email="vaibhav@example.com",
        subscription=subscription,
    )

    # devices
    ipad = Device(device_id="d1", name="iPad", device_type=DeviceType.TABLET)
    tv = Device(device_id="d2", name="Living Room TV", device_type=DeviceType.TV)
    user.add_device(ipad)
    user.add_device(tv)

    # content
    movie = Movie(
        content_id="m1",
        title="The Tomorrow War",
        description="Sci fi action movie",
        genre="Sci-Fi",
        release_year=2021,
        rating=7.0,
        duration_minutes=138,
        content_type=ContentType.MOVIE,
    )

    ep1 = Episode(
        content_id="e1",
        title="Reacher Episode 1",
        description="Pilot episode",
        genre="Action",
        release_year=2022,
        rating=8.2,
        duration_minutes=48,
        content_type=ContentType.EPISODE,
        episode_number=1,
    )

    ep2 = Episode(
        content_id="e2",
        title="Reacher Episode 2",
        description="Second episode",
        genre="Action",
        release_year=2022,
        rating=8.1,
        duration_minutes=46,
        content_type=ContentType.EPISODE,
        episode_number=2,
    )

    season1 = Season(season_number=1)
    season1.add_episode(ep1)
    season1.add_episode(ep2)

    series = Series(
        content_id="s1",
        title="Reacher",
        description="Action thriller series",
        genre="Action",
        release_year=2022,
        rating=8.5,
        duration_minutes=0,
        content_type=ContentType.SERIES,
        seasons=[season1],
    )

    # catalog
    catalog = Catalog()
    catalog.add_content(movie)
    catalog.add_content(series)
    catalog.add_content(ep1)
    catalog.add_content(ep2)

    # service
    service = PrimeVideoService(catalog)

    # watchlist
    service.add_to_watchlist(user, "m1")
    service.add_to_watchlist(user, "s1")

    print("=== Watchlist ===")
    for item in service.get_watchlist(user):
        print(item.content.title)

    # search
    print("\n=== Search 'reach' ===")
    for content in service.search("reach"):
        print(content.title, content.content_type.value)

    # playback
    print("\n=== Start Playback ===")
    session = service.start_playback(user, "m1", "d1", VideoQuality.HD)
    print("Session ID:", session.session_id)
    print("Start position:", session.current_position_seconds)

    service.update_playback_progress(session.session_id, 1200)
    service.stop_playback(session.session_id)

    print("\n=== Watch History ===")
    history = service.get_watch_history(user)
    movie_progress = history.get_progress("m1")
    if movie_progress:
        print("Movie progress seconds:", movie_progress.progress_seconds)

    print("\n=== Resume Playback ===")
    resumed_session = service.start_playback(user, "m1", "d2", VideoQuality.UHD)
    print("Resumed from:", resumed_session.current_position_seconds)

    service.update_playback_progress(resumed_session.session_id, 8280, completed=True)

    print("\n=== Continue Watching ===")
    continue_items = service.get_continue_watching(user)
    for item in continue_items:
        print(item.content_id, item.progress_seconds)
```

**Advanced code**


```python
from __future__ import annotations

from abc import ABC
from dataclasses import dataclass, field
from datetime import datetime, timedelta
from enum import Enum
from typing import Dict, List, Optional, Set
import uuid


# =========================
# Enums
# =========================

class SubscriptionStatus(str, Enum):
    ACTIVE = "ACTIVE"
    EXPIRED = "EXPIRED"
    CANCELLED = "CANCELLED"


class ContentType(str, Enum):
    MOVIE = "MOVIE"
    SERIES = "SERIES"
    EPISODE = "EPISODE"


class DeviceType(str, Enum):
    MOBILE = "MOBILE"
    TABLET = "TABLET"
    WEB = "WEB"
    TV = "TV"


class VideoQuality(str, Enum):
    SD = "SD"
    HD = "HD"
    UHD = "UHD"


class SessionStatus(str, Enum):
    ACTIVE = "ACTIVE"
    STOPPED = "STOPPED"
    COMPLETED = "COMPLETED"


class AccessType(str, Enum):
    INCLUDED = "INCLUDED"
    RENT = "RENT"
    BUY = "BUY"


class MaturityRating(str, Enum):
    KIDS = "KIDS"
    TEEN = "TEEN"
    ADULT = "ADULT"


# =========================
# Subscription / Account / Profile / Device
# =========================

@dataclass
class Subscription:
    subscription_id: str
    status: SubscriptionStatus
    start_date: datetime
    end_date: datetime
    max_concurrent_streams: int = 3

    def is_active(self) -> bool:
        now = datetime.now()
        return self.status == SubscriptionStatus.ACTIVE and self.start_date <= now <= self.end_date

    def expire_if_needed(self) -> None:
        if datetime.now() > self.end_date and self.status == SubscriptionStatus.ACTIVE:
            self.status = SubscriptionStatus.EXPIRED


@dataclass(frozen=True)
class Device:
    device_id: str
    name: str
    device_type: DeviceType


@dataclass
class Profile:
    profile_id: str
    name: str
    maturity_limit: MaturityRating
    is_kids_profile: bool = False


@dataclass
class Account:
    account_id: str
    name: str
    email: str
    region: str
    subscription: Subscription
    devices: List[Device] = field(default_factory=list)
    profiles: List[Profile] = field(default_factory=list)

    def add_device(self, device: Device) -> None:
        self.devices.append(device)

    def get_device(self, device_id: str) -> Optional[Device]:
        for device in self.devices:
            if device.device_id == device_id:
                return device
        return None

    def add_profile(self, profile: Profile) -> None:
        self.profiles.append(profile)

    def get_profile(self, profile_id: str) -> Optional[Profile]:
        for profile in self.profiles:
            if profile.profile_id == profile_id:
                return profile
        return None


# =========================
# Content Hierarchy
# =========================

@dataclass
class Content(ABC):
    content_id: str
    title: str
    description: str
    genre: str
    release_year: int
    rating: float
    duration_minutes: int
    content_type: ContentType
    maturity_rating: MaturityRating
    allowed_regions: Set[str]
    access_type: AccessType


@dataclass
class Movie(Content):
    pass


@dataclass
class Episode(Content):
    episode_number: int = 0


@dataclass
class Season:
    season_number: int
    episodes: List[Episode] = field(default_factory=list)

    def add_episode(self, episode: Episode) -> None:
        self.episodes.append(episode)


@dataclass
class Series(Content):
    seasons: List[Season] = field(default_factory=list)

    def add_season(self, season: Season) -> None:
        self.seasons.append(season)


# =========================
# Watchlist / Watch History
# =========================

@dataclass
class WatchlistItem:
    content: Content
    added_at: datetime = field(default_factory=datetime.now)


@dataclass
class WatchProgress:
    content_id: str
    progress_seconds: int
    completed: bool = False
    updated_at: datetime = field(default_factory=datetime.now)


@dataclass
class WatchHistory:
    profile_id: str
    progress_map: Dict[str, WatchProgress] = field(default_factory=dict)

    def update_progress(self, content_id: str, progress_seconds: int, completed: bool = False) -> None:
        self.progress_map[content_id] = WatchProgress(
            content_id=content_id,
            progress_seconds=progress_seconds,
            completed=completed,
            updated_at=datetime.now(),
        )

    def get_progress(self, content_id: str) -> Optional[WatchProgress]:
        return self.progress_map.get(content_id)

    def get_continue_watching(self) -> List[WatchProgress]:
        return [p for p in self.progress_map.values() if not p.completed]


# =========================
# Purchase / Rental
# =========================

@dataclass
class PurchaseRecord:
    purchase_id: str
    account_id: str
    content_id: str
    purchased_at: datetime


@dataclass
class RentalRecord:
    rental_id: str
    account_id: str
    content_id: str
    rented_at: datetime
    expires_at: datetime

    def is_active(self) -> bool:
        return datetime.now() <= self.expires_at


# =========================
# Playback
# =========================

@dataclass
class PlaybackSession:
    session_id: str
    account: Account
    profile: Profile
    content: Content
    device: Device
    started_at: datetime
    quality: VideoQuality
    current_position_seconds: int = 0
    status: SessionStatus = SessionStatus.ACTIVE

    def stop(self) -> None:
        self.status = SessionStatus.STOPPED

    def complete(self) -> None:
        self.status = SessionStatus.COMPLETED

    def is_active(self) -> bool:
        return self.status == SessionStatus.ACTIVE


# =========================
# Catalog
# =========================

class Catalog:
    def __init__(self):
        self.contents: Dict[str, Content] = {}

    def add_content(self, content: Content) -> None:
        self.contents[content.content_id] = content

    def get_content_by_id(self, content_id: str) -> Optional[Content]:
        return self.contents.get(content_id)

    def search_by_title(self, query: str) -> List[Content]:
        q = query.lower()
        return [c for c in self.contents.values() if q in c.title.lower()]

    def filter_by_genre(self, genre: str) -> List[Content]:
        return [c for c in self.contents.values() if c.genre.lower() == genre.lower()]

    def filter_by_type(self, content_type: ContentType) -> List[Content]:
        return [c for c in self.contents.values() if c.content_type == content_type]


# =========================
# Repositories
# =========================

class WatchlistRepository:
    def __init__(self):
        self.watchlists: Dict[str, List[WatchlistItem]] = {}

    def add_item(self, profile_id: str, item: WatchlistItem) -> None:
        self.watchlists.setdefault(profile_id, []).append(item)

    def remove_item(self, profile_id: str, content_id: str) -> None:
        self.watchlists[profile_id] = [
            item for item in self.watchlists.get(profile_id, [])
            if item.content.content_id != content_id
        ]

    def get_watchlist(self, profile_id: str) -> List[WatchlistItem]:
        return list(self.watchlists.get(profile_id, []))

    def exists(self, profile_id: str, content_id: str) -> bool:
        return any(item.content.content_id == content_id for item in self.watchlists.get(profile_id, []))


class WatchHistoryRepository:
    def __init__(self):
        self.histories: Dict[str, WatchHistory] = {}

    def get_or_create(self, profile_id: str) -> WatchHistory:
        if profile_id not in self.histories:
            self.histories[profile_id] = WatchHistory(profile_id=profile_id)
        return self.histories[profile_id]


class PlaybackSessionRepository:
    def __init__(self):
        self.sessions: Dict[str, PlaybackSession] = {}

    def save(self, session: PlaybackSession) -> None:
        self.sessions[session.session_id] = session

    def get_by_id(self, session_id: str) -> Optional[PlaybackSession]:
        return self.sessions.get(session_id)

    def get_active_sessions_for_account(self, account_id: str) -> List[PlaybackSession]:
        return [
            s for s in self.sessions.values()
            if s.account.account_id == account_id and s.is_active()
        ]


class PurchaseRepository:
    def __init__(self):
        self.purchases: Dict[str, List[PurchaseRecord]] = {}

    def add_purchase(self, record: PurchaseRecord) -> None:
        self.purchases.setdefault(record.account_id, []).append(record)

    def has_purchase(self, account_id: str, content_id: str) -> bool:
        return any(r.content_id == content_id for r in self.purchases.get(account_id, []))


class RentalRepository:
    def __init__(self):
        self.rentals: Dict[str, List[RentalRecord]] = {}

    def add_rental(self, record: RentalRecord) -> None:
        self.rentals.setdefault(record.account_id, []).append(record)

    def has_active_rental(self, account_id: str, content_id: str) -> bool:
        return any(r.content_id == content_id and r.is_active() for r in self.rentals.get(account_id, []))


# =========================
# Prime Video Service
# =========================

class PrimeVideoService:
    def __init__(self, catalog: Catalog):
        self.catalog = catalog
        self.watchlist_repository = WatchlistRepository()
        self.watch_history_repository = WatchHistoryRepository()
        self.playback_session_repository = PlaybackSessionRepository()
        self.purchase_repository = PurchaseRepository()
        self.rental_repository = RentalRepository()

    # ---------- Content ----------
    def search(self, query: str) -> List[Content]:
        return self.catalog.search_by_title(query)

    def browse_by_genre(self, genre: str) -> List[Content]:
        return self.catalog.filter_by_genre(genre)

    def browse_movies(self) -> List[Content]:
        return self.catalog.filter_by_type(ContentType.MOVIE)

    def browse_series(self) -> List[Content]:
        return self.catalog.filter_by_type(ContentType.SERIES)

    # ---------- Watchlist ----------
    def add_to_watchlist(self, profile: Profile, content_id: str) -> None:
        content = self.catalog.get_content_by_id(content_id)
        if content is None:
            raise ValueError("Content not found")
        if self.watchlist_repository.exists(profile.profile_id, content_id):
            return
        self.watchlist_repository.add_item(profile.profile_id, WatchlistItem(content=content))

    def get_watchlist(self, profile: Profile) -> List[WatchlistItem]:
        return self.watchlist_repository.get_watchlist(profile.profile_id)

    # ---------- Access Checks ----------
    def _check_region_access(self, account: Account, content: Content) -> None:
        if account.region not in content.allowed_regions:
            raise ValueError("Content not available in this region")

    def _check_parental_control(self, profile: Profile, content: Content) -> None:
        order = {
            MaturityRating.KIDS: 1,
            MaturityRating.TEEN: 2,
            MaturityRating.ADULT: 3,
        }
        if order[content.maturity_rating] > order[profile.maturity_limit]:
            raise ValueError("Blocked by parental control")

        if profile.is_kids_profile and content.maturity_rating != MaturityRating.KIDS:
            raise ValueError("Kids profile cannot access this content")

    def _check_content_access_policy(self, account: Account, content: Content) -> None:
        account.subscription.expire_if_needed()

        if content.access_type == AccessType.INCLUDED:
            if not account.subscription.is_active():
                raise ValueError("Active subscription required")

        elif content.access_type == AccessType.RENT:
            if not self.rental_repository.has_active_rental(account.account_id, content.content_id):
                raise ValueError("Active rental required")

        elif content.access_type == AccessType.BUY:
            if not self.purchase_repository.has_purchase(account.account_id, content.content_id):
                raise ValueError("Purchase required")

    def _check_stream_limit(self, account: Account) -> None:
        active_sessions = self.playback_session_repository.get_active_sessions_for_account(account.account_id)
        if len(active_sessions) >= account.subscription.max_concurrent_streams:
            raise ValueError("Concurrent stream limit reached")

    # ---------- Purchase / Rent ----------
    def buy_content(self, account: Account, content_id: str) -> PurchaseRecord:
        content = self.catalog.get_content_by_id(content_id)
        if content is None:
            raise ValueError("Content not found")

        record = PurchaseRecord(
            purchase_id=str(uuid.uuid4()),
            account_id=account.account_id,
            content_id=content_id,
            purchased_at=datetime.now(),
        )
        self.purchase_repository.add_purchase(record)
        return record

    def rent_content(self, account: Account, content_id: str, rental_hours: int = 48) -> RentalRecord:
        content = self.catalog.get_content_by_id(content_id)
        if content is None:
            raise ValueError("Content not found")

        record = RentalRecord(
            rental_id=str(uuid.uuid4()),
            account_id=account.account_id,
            content_id=content_id,
            rented_at=datetime.now(),
            expires_at=datetime.now() + timedelta(hours=rental_hours),
        )
        self.rental_repository.add_rental(record)
        return record

    # ---------- Playback ----------
    def start_playback(
        self,
        account: Account,
        profile_id: str,
        content_id: str,
        device_id: str,
        quality: VideoQuality,
    ) -> PlaybackSession:
        content = self.catalog.get_content_by_id(content_id)
        if content is None:
            raise ValueError("Content not found")

        profile = account.get_profile(profile_id)
        if profile is None:
            raise ValueError("Profile not found")

        device = account.get_device(device_id)
        if device is None:
            raise ValueError("Device not found")

        self._check_region_access(account, content)
        self._check_parental_control(profile, content)
        self._check_content_access_policy(account, content)
        self._check_stream_limit(account)

        history = self.watch_history_repository.get_or_create(profile.profile_id)
        progress = history.get_progress(content_id)
        start_position = progress.progress_seconds if progress else 0

        session = PlaybackSession(
            session_id=str(uuid.uuid4()),
            account=account,
            profile=profile,
            content=content,
            device=device,
            started_at=datetime.now(),
            quality=quality,
            current_position_seconds=start_position,
        )
        self.playback_session_repository.save(session)
        return session

    def update_playback_progress(self, session_id: str, progress_seconds: int, completed: bool = False) -> None:
        session = self.playback_session_repository.get_by_id(session_id)
        if session is None or not session.is_active():
            raise ValueError("Invalid or inactive session")

        session.current_position_seconds = progress_seconds
        history = self.watch_history_repository.get_or_create(session.profile.profile_id)
        history.update_progress(session.content.content_id, progress_seconds, completed)

        if completed:
            session.complete()

        self.playback_session_repository.save(session)

    def stop_playback(self, session_id: str) -> None:
        session = self.playback_session_repository.get_by_id(session_id)
        if session is None:
            raise ValueError("Session not found")

        history = self.watch_history_repository.get_or_create(session.profile.profile_id)
        history.update_progress(
            session.content.content_id,
            session.current_position_seconds,
            False,
        )
        session.stop()
        self.playback_session_repository.save(session)

    def get_watch_history(self, profile: Profile) -> WatchHistory:
        return self.watch_history_repository.get_or_create(profile.profile_id)


# =========================
# Demo
# =========================

if __name__ == "__main__":
    subscription = Subscription(
        subscription_id="sub1",
        status=SubscriptionStatus.ACTIVE,
        start_date=datetime.now() - timedelta(days=10),
        end_date=datetime.now() + timedelta(days=20),
        max_concurrent_streams=2,
    )

    account = Account(
        account_id="acc1",
        name="Vaibhav",
        email="vaibhav@example.com",
        region="US",
        subscription=subscription,
    )

    account.add_device(Device("d1", "iPad", DeviceType.TABLET))
    account.add_device(Device("d2", "TV", DeviceType.TV))

    adult_profile = Profile("p1", "Main", MaturityRating.ADULT, False)
    kids_profile = Profile("p2", "Kids", MaturityRating.KIDS, True)
    account.add_profile(adult_profile)
    account.add_profile(kids_profile)

    included_movie = Movie(
        content_id="m1",
        title="Included Movie",
        description="Prime included title",
        genre="Action",
        release_year=2024,
        rating=8.0,
        duration_minutes=120,
        content_type=ContentType.MOVIE,
        maturity_rating=MaturityRating.TEEN,
        allowed_regions={"US", "IN"},
        access_type=AccessType.INCLUDED,
    )

    rental_movie = Movie(
        content_id="m2",
        title="Rental Movie",
        description="Rent only title",
        genre="Drama",
        release_year=2023,
        rating=7.5,
        duration_minutes=110,
        content_type=ContentType.MOVIE,
        maturity_rating=MaturityRating.ADULT,
        allowed_regions={"US"},
        access_type=AccessType.RENT,
    )

    bought_movie = Movie(
        content_id="m3",
        title="Buy Movie",
        description="Buy only title",
        genre="Sci-Fi",
        release_year=2022,
        rating=8.7,
        duration_minutes=130,
        content_type=ContentType.MOVIE,
        maturity_rating=MaturityRating.ADULT,
        allowed_regions={"US"},
        access_type=AccessType.BUY,
    )

    catalog = Catalog()
    catalog.add_content(included_movie)
    catalog.add_content(rental_movie)
    catalog.add_content(bought_movie)

    service = PrimeVideoService(catalog)

    service.add_to_watchlist(adult_profile, "m1")
    print("Watchlist:", [item.content.title for item in service.get_watchlist(adult_profile)])

    session1 = service.start_playback(account, "p1", "m1", "d1", VideoQuality.HD)
    print("Started included content:", session1.session_id)

    service.rent_content(account, "m2")
    session2 = service.start_playback(account, "p1", "m2", "d2", VideoQuality.HD)
    print("Started rented content:", session2.session_id)

    try:
        service.start_playback(account, "p2", "m2", "d1", VideoQuality.SD)
    except Exception as e:
        print("Kids profile blocked:", str(e))

    service.stop_playback(session1.session_id)

    service.buy_content(account, "m3")
    session3 = service.start_playback(account, "p1", "m3", "d1", VideoQuality.UHD)
    print("Started purchased content:", session3.session_id)
```

you the design patterns, SOLID principles, and OOP concepts used in this Prime Video LLD in markdown code.