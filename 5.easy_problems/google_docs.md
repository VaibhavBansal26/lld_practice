# 15. Google Docs

Code

```python
from __future__ import annotations

from abc import ABC, abstractmethod
from dataclasses import dataclass, field
from datetime import datetime
from enum import Enum
from typing import Dict, List, Optional
import copy
import uuid


# =========================
# Enums
# =========================

class PermissionRole(str, Enum):
    OWNER = "OWNER"
    EDITOR = "EDITOR"
    COMMENTER = "COMMENTER"
    VIEWER = "VIEWER"


class BlockType(str, Enum):
    PARAGRAPH = "PARAGRAPH"
    HEADING = "HEADING"
    BULLET_ITEM = "BULLET_ITEM"
    NUMBERED_ITEM = "NUMBERED_ITEM"


class CommentStatus(str, Enum):
    OPEN = "OPEN"
    RESOLVED = "RESOLVED"


# =========================
# Core Entities
# =========================

@dataclass(frozen=True)
class User:
    user_id: str
    name: str
    email: str


@dataclass
class Style:
    bold: bool = False
    italic: bool = False
    underline: bool = False
    font_size: int = 12
    font_family: str = "Arial"
    color: str = "#000000"


@dataclass
class TextRun:
    text: str
    style: Style


@dataclass
class Block:
    block_id: str
    block_type: BlockType
    runs: List[TextRun] = field(default_factory=list)

    def get_plain_text(self) -> str:
        return "".join(run.text for run in self.runs)


@dataclass
class Permission:
    user: User
    role: PermissionRole


@dataclass
class Comment:
    comment_id: str
    author: User
    block_id: str
    start_offset: int
    end_offset: int
    text: str
    status: CommentStatus = CommentStatus.OPEN
    created_at: datetime = field(default_factory=datetime.now)

    def resolve(self) -> None:
        self.status = CommentStatus.RESOLVED


@dataclass
class Version:
    version_id: str
    created_by: User
    snapshot_blocks: List[Block]
    created_at: datetime = field(default_factory=datetime.now)


@dataclass
class CursorPosition:
    block_id: str
    offset: int


@dataclass
class CollaborationSession:
    session_id: str
    user: User
    cursor_position: CursorPosition
    joined_at: datetime = field(default_factory=datetime.now)
    active: bool = True

    def leave(self) -> None:
        self.active = False


@dataclass
class Document:
    doc_id: str
    title: str
    owner: User
    blocks: List[Block] = field(default_factory=list)
    permissions: Dict[str, Permission] = field(default_factory=dict)
    comments: List[Comment] = field(default_factory=list)
    versions: List[Version] = field(default_factory=list)
    active_sessions: Dict[str, CollaborationSession] = field(default_factory=dict)

    def add_permission(self, permission: Permission) -> None:
        self.permissions[permission.user.user_id] = permission

    def get_user_role(self, user_id: str) -> Optional[PermissionRole]:
        if user_id == self.owner.user_id:
            return PermissionRole.OWNER
        permission = self.permissions.get(user_id)
        return permission.role if permission else None

    def can_view(self, user: User) -> bool:
        role = self.get_user_role(user.user_id)
        return role in {
            PermissionRole.OWNER,
            PermissionRole.EDITOR,
            PermissionRole.COMMENTER,
            PermissionRole.VIEWER,
        }

    def can_comment(self, user: User) -> bool:
        role = self.get_user_role(user.user_id)
        return role in {
            PermissionRole.OWNER,
            PermissionRole.EDITOR,
            PermissionRole.COMMENTER,
        }

    def can_edit(self, user: User) -> bool:
        role = self.get_user_role(user.user_id)
        return role in {
            PermissionRole.OWNER,
            PermissionRole.EDITOR,
        }


# =========================
# Edit Operations
# =========================

class EditOperation(ABC):
    @abstractmethod
    def apply(self, document: Document) -> None:
        pass


class InsertTextOperation(EditOperation):
    def __init__(self, block_id: str, offset: int, text: str, style: Style):
        self.block_id = block_id
        self.offset = offset
        self.text = text
        self.style = style

    def apply(self, document: Document) -> None:
        block = next((b for b in document.blocks if b.block_id == self.block_id), None)
        if block is None:
            raise ValueError("Block not found")

        full_text = block.get_plain_text()
        if self.offset < 0 or self.offset > len(full_text):
            raise ValueError("Invalid insert offset")

        new_text = full_text[:self.offset] + self.text + full_text[self.offset:]
        block.runs = [TextRun(new_text, self.style)]


class DeleteTextOperation(EditOperation):
    def __init__(self, block_id: str, start_offset: int, end_offset: int, style: Style):
        self.block_id = block_id
        self.start_offset = start_offset
        self.end_offset = end_offset
        self.style = style

    def apply(self, document: Document) -> None:
        block = next((b for b in document.blocks if b.block_id == self.block_id), None)
        if block is None:
            raise ValueError("Block not found")

        full_text = block.get_plain_text()
        if (
            self.start_offset < 0
            or self.end_offset > len(full_text)
            or self.start_offset > self.end_offset
        ):
            raise ValueError("Invalid delete range")

        new_text = full_text[:self.start_offset] + full_text[self.end_offset:]
        block.runs = [TextRun(new_text, self.style)]


class ApplyStyleOperation(EditOperation):
    def __init__(self, block_id: str, style: Style):
        self.block_id = block_id
        self.style = style

    def apply(self, document: Document) -> None:
        block = next((b for b in document.blocks if b.block_id == self.block_id), None)
        if block is None:
            raise ValueError("Block not found")

        text = block.get_plain_text()
        block.runs = [TextRun(text, self.style)]


# =========================
# Document Service
# =========================

class DocumentService:
    def __init__(self):
        self.documents: Dict[str, Document] = {}

    def create_document(self, owner: User, title: str) -> Document:
        doc = Document(
            doc_id=str(uuid.uuid4()),
            title=title,
            owner=owner,
        )
        self.documents[doc.doc_id] = doc
        return doc

    def get_document(self, doc_id: str) -> Document:
        if doc_id not in self.documents:
            raise ValueError("Document not found")
        return self.documents[doc_id]

    def add_block(
        self,
        doc_id: str,
        user: User,
        block_type: BlockType,
        text: str,
        style: Style,
    ) -> Block:
        doc = self.get_document(doc_id)
        if not doc.can_edit(user):
            raise PermissionError("User cannot edit")

        block = Block(
            block_id=str(uuid.uuid4()),
            block_type=block_type,
            runs=[TextRun(text, style)],
        )
        doc.blocks.append(block)
        return block

    def apply_operation(self, doc_id: str, user: User, operation: EditOperation) -> None:
        doc = self.get_document(doc_id)
        if not doc.can_edit(user):
            raise PermissionError("User cannot edit")
        operation.apply(doc)

    def add_comment(
        self,
        doc_id: str,
        user: User,
        block_id: str,
        start_offset: int,
        end_offset: int,
        text: str,
    ) -> Comment:
        doc = self.get_document(doc_id)
        if not doc.can_comment(user):
            raise PermissionError("User cannot comment")

        block = next((b for b in doc.blocks if b.block_id == block_id), None)
        if block is None:
            raise ValueError("Block not found")

        full_text = block.get_plain_text()
        if start_offset < 0 or end_offset > len(full_text) or start_offset > end_offset:
            raise ValueError("Invalid comment range")

        comment = Comment(
            comment_id=str(uuid.uuid4()),
            author=user,
            block_id=block_id,
            start_offset=start_offset,
            end_offset=end_offset,
            text=text,
        )
        doc.comments.append(comment)
        return comment

    def resolve_comment(self, doc_id: str, user: User, comment_id: str) -> None:
        doc = self.get_document(doc_id)
        if not doc.can_comment(user):
            raise PermissionError("User cannot resolve comments")

        comment = next((c for c in doc.comments if c.comment_id == comment_id), None)
        if comment is None:
            raise ValueError("Comment not found")

        comment.resolve()

    def share_document(
        self,
        doc_id: str,
        owner: User,
        target_user: User,
        role: PermissionRole,
    ) -> None:
        doc = self.get_document(doc_id)
        if doc.owner.user_id != owner.user_id:
            raise PermissionError("Only owner can share")

        doc.add_permission(Permission(target_user, role))

    def save_version(self, doc_id: str, user: User) -> Version:
        doc = self.get_document(doc_id)
        if not doc.can_edit(user):
            raise PermissionError("User cannot save version")

        snapshot = copy.deepcopy(doc.blocks)
        version = Version(
            version_id=str(uuid.uuid4()),
            created_by=user,
            snapshot_blocks=snapshot,
        )
        doc.versions.append(version)
        return version

    def get_versions(self, doc_id: str, user: User) -> List[Version]:
        doc = self.get_document(doc_id)
        if not doc.can_view(user):
            raise PermissionError("User cannot view versions")
        return list(doc.versions)

    def restore_version(self, doc_id: str, user: User, version_id: str) -> None:
        doc = self.get_document(doc_id)
        if not doc.can_edit(user):
            raise PermissionError("User cannot restore version")

        version = next((v for v in doc.versions if v.version_id == version_id), None)
        if version is None:
            raise ValueError("Version not found")

        doc.blocks = copy.deepcopy(version.snapshot_blocks)

    def join_session(
        self,
        doc_id: str,
        user: User,
        cursor_position: CursorPosition,
    ) -> CollaborationSession:
        doc = self.get_document(doc_id)
        if not doc.can_view(user):
            raise PermissionError("User cannot access document")

        session = CollaborationSession(
            session_id=str(uuid.uuid4()),
            user=user,
            cursor_position=cursor_position,
        )
        doc.active_sessions[user.user_id] = session
        return session

    def update_cursor(
        self,
        doc_id: str,
        user: User,
        cursor_position: CursorPosition,
    ) -> None:
        doc = self.get_document(doc_id)
        session = doc.active_sessions.get(user.user_id)
        if session is None:
            raise ValueError("Session not found")
        session.cursor_position = cursor_position

    def leave_session(self, doc_id: str, user: User) -> None:
        doc = self.get_document(doc_id)
        session = doc.active_sessions.get(user.user_id)
        if session:
            session.leave()
            del doc.active_sessions[user.user_id]

    def get_active_sessions(self, doc_id: str, user: User) -> List[CollaborationSession]:
        doc = self.get_document(doc_id)
        if not doc.can_view(user):
            raise PermissionError("User cannot view active sessions")
        return list(doc.active_sessions.values())


# =========================
# Demo
# =========================

if __name__ == "__main__":
    service = DocumentService()

    owner = User("u1", "Alice", "alice@example.com")
    editor = User("u2", "Bob", "bob@example.com")
    commenter = User("u3", "Charlie", "charlie@example.com")
    viewer = User("u4", "David", "david@example.com")

    doc = service.create_document(owner, "Google Docs LLD")

    service.share_document(doc.doc_id, owner, editor, PermissionRole.EDITOR)
    service.share_document(doc.doc_id, owner, commenter, PermissionRole.COMMENTER)
    service.share_document(doc.doc_id, owner, viewer, PermissionRole.VIEWER)

    normal_style = Style()
    heading_style = Style(bold=True, font_size=18)
    emphasis_style = Style(italic=True, color="#1a73e8")

    heading = service.add_block(
        doc.doc_id,
        owner,
        BlockType.HEADING,
        "System Design Notes",
        heading_style,
    )

    para = service.add_block(
        doc.doc_id,
        owner,
        BlockType.PARAGRAPH,
        "Initial content here.",
        normal_style,
    )

    service.apply_operation(
        doc.doc_id,
        editor,
        InsertTextOperation(
            para.block_id,
            len("Initial content here."),
            " More text added by editor.",
            normal_style,
        ),
    )

    service.apply_operation(
        doc.doc_id,
        editor,
        ApplyStyleOperation(para.block_id, emphasis_style),
    )

    comment = service.add_comment(
        doc.doc_id,
        commenter,
        para.block_id,
        0,
        7,
        "Can we improve this intro?",
    )

    version = service.save_version(doc.doc_id, owner)

    session_owner = service.join_session(
        doc.doc_id,
        owner,
        CursorPosition(block_id=heading.block_id, offset=5),
    )

    session_editor = service.join_session(
        doc.doc_id,
        editor,
        CursorPosition(block_id=para.block_id, offset=10),
    )

    service.update_cursor(
        doc.doc_id,
        editor,
        CursorPosition(block_id=para.block_id, offset=20),
    )

    print("Document title:", doc.title)
    print("Owner:", doc.owner.name)
    print("Blocks:", len(doc.blocks))
    print("Paragraph text:", para.get_plain_text())
    print("Comments:", len(doc.comments))
    print("Versions:", len(doc.versions))
    print("Active sessions:", len(doc.active_sessions))
    print("Saved version id:", version.version_id)
    print("Comment status:", comment.status.value)

    service.resolve_comment(doc.doc_id, owner, comment.comment_id)
    print("Resolved comment status:", comment.status.value)

    service.leave_session(doc.doc_id, editor)
    print("Active sessions after editor leaves:", len(doc.active_sessions))

    print("\nVisible to viewer:", service.get_document(doc.doc_id).can_view(viewer))
    print("Viewer can edit:", service.get_document(doc.doc_id).can_edit(viewer))

    print("\nDocument content by block:")
    for block in doc.blocks:
        print(f"{block.block_type.value}: {block.get_plain_text()}")
```

**Advanced Code**

Undo / redo

Comment replies

Selection range instead of only cursor

Suggestion mode

OT / CRDT collaboration engine placeholder

```python
from __future__ import annotations

from abc import ABC, abstractmethod
from dataclasses import dataclass, field
from datetime import datetime
from enum import Enum
from typing import Dict, List, Optional
import copy
import uuid


# =========================
# Enums
# =========================

class PermissionRole(str, Enum):
    OWNER = "OWNER"
    EDITOR = "EDITOR"
    COMMENTER = "COMMENTER"
    VIEWER = "VIEWER"


class BlockType(str, Enum):
    PARAGRAPH = "PARAGRAPH"
    HEADING = "HEADING"
    BULLET_ITEM = "BULLET_ITEM"
    NUMBERED_ITEM = "NUMBERED_ITEM"


class CommentStatus(str, Enum):
    OPEN = "OPEN"
    RESOLVED = "RESOLVED"


class EditMode(str, Enum):
    EDIT = "EDIT"
    SUGGEST = "SUGGEST"


class SuggestionStatus(str, Enum):
    PENDING = "PENDING"
    ACCEPTED = "ACCEPTED"
    REJECTED = "REJECTED"


# =========================
# Core Entities
# =========================

@dataclass(frozen=True)
class User:
    user_id: str
    name: str
    email: str


@dataclass
class Style:
    bold: bool = False
    italic: bool = False
    underline: bool = False
    font_size: int = 12
    font_family: str = "Arial"
    color: str = "#000000"


@dataclass
class TextRun:
    text: str
    style: Style


@dataclass
class Block:
    block_id: str
    block_type: BlockType
    runs: List[TextRun] = field(default_factory=list)

    def get_plain_text(self) -> str:
        return "".join(run.text for run in self.runs)

    def set_plain_text(self, text: str, style: Optional[Style] = None) -> None:
        style = style or Style()
        self.runs = [TextRun(text, style)]


@dataclass
class Permission:
    user: User
    role: PermissionRole


@dataclass
class CommentReply:
    reply_id: str
    author: User
    text: str
    created_at: datetime = field(default_factory=datetime.now)


@dataclass
class Comment:
    comment_id: str
    author: User
    block_id: str
    start_offset: int
    end_offset: int
    text: str
    status: CommentStatus = CommentStatus.OPEN
    replies: List[CommentReply] = field(default_factory=list)
    created_at: datetime = field(default_factory=datetime.now)

    def add_reply(self, author: User, text: str) -> CommentReply:
        reply = CommentReply(
            reply_id=str(uuid.uuid4()),
            author=author,
            text=text,
        )
        self.replies.append(reply)
        return reply

    def resolve(self) -> None:
        self.status = CommentStatus.RESOLVED


@dataclass
class Version:
    version_id: str
    created_by: User
    snapshot_blocks: List[Block]
    created_at: datetime = field(default_factory=datetime.now)


@dataclass
class SelectionRange:
    block_id: str
    start_offset: int
    end_offset: int

    def is_cursor_only(self) -> bool:
        return self.start_offset == self.end_offset


@dataclass
class CollaborationSession:
    session_id: str
    user: User
    selection_range: SelectionRange
    joined_at: datetime = field(default_factory=datetime.now)
    active: bool = True

    def leave(self) -> None:
        self.active = False


@dataclass
class Suggestion:
    suggestion_id: str
    suggested_by: User
    operation: "EditOperation"
    status: SuggestionStatus = SuggestionStatus.PENDING
    created_at: datetime = field(default_factory=datetime.now)


@dataclass
class Document:
    doc_id: str
    title: str
    owner: User
    blocks: List[Block] = field(default_factory=list)
    permissions: Dict[str, Permission] = field(default_factory=dict)
    comments: List[Comment] = field(default_factory=list)
    versions: List[Version] = field(default_factory=list)
    active_sessions: Dict[str, CollaborationSession] = field(default_factory=dict)
    suggestions: List[Suggestion] = field(default_factory=list)

    def add_permission(self, permission: Permission) -> None:
        self.permissions[permission.user.user_id] = permission

    def get_user_role(self, user_id: str) -> Optional[PermissionRole]:
        if user_id == self.owner.user_id:
            return PermissionRole.OWNER
        permission = self.permissions.get(user_id)
        return permission.role if permission else None

    def can_view(self, user: User) -> bool:
        role = self.get_user_role(user.user_id)
        return role in {
            PermissionRole.OWNER,
            PermissionRole.EDITOR,
            PermissionRole.COMMENTER,
            PermissionRole.VIEWER,
        }

    def can_comment(self, user: User) -> bool:
        role = self.get_user_role(user.user_id)
        return role in {
            PermissionRole.OWNER,
            PermissionRole.EDITOR,
            PermissionRole.COMMENTER,
        }

    def can_edit(self, user: User) -> bool:
        role = self.get_user_role(user.user_id)
        return role in {
            PermissionRole.OWNER,
            PermissionRole.EDITOR,
        }


# =========================
# Collaboration Engine Placeholder
# OT / CRDT ready abstraction
# =========================

class CollaborationEngine(ABC):
    @abstractmethod
    def transform_and_apply(
        self,
        document: Document,
        operation: "EditOperation",
    ) -> None:
        pass


class SimpleCollaborationEngine(CollaborationEngine):
    """
    Interview placeholder:
    In production, this would be replaced by OT or CRDT logic.
    Right now, it simply applies the operation directly.
    """

    def transform_and_apply(
        self,
        document: Document,
        operation: "EditOperation",
    ) -> None:
        operation.apply(document)


# =========================
# Edit Operations
# Command style + undo support
# =========================

class EditOperation(ABC):
    @abstractmethod
    def apply(self, document: Document) -> None:
        pass

    @abstractmethod
    def undo(self, document: Document) -> None:
        pass


class InsertTextOperation(EditOperation):
    def __init__(self, block_id: str, offset: int, text: str, style: Style):
        self.block_id = block_id
        self.offset = offset
        self.text = text
        self.style = style
        self.previous_text: Optional[str] = None
        self.previous_style: Optional[Style] = None

    def apply(self, document: Document) -> None:
        block = _find_block(document, self.block_id)
        full_text = block.get_plain_text()

        if self.offset < 0 or self.offset > len(full_text):
            raise ValueError("Invalid insert offset")

        self.previous_text = full_text
        self.previous_style = block.runs[0].style if block.runs else Style()

        new_text = full_text[:self.offset] + self.text + full_text[self.offset:]
        block.set_plain_text(new_text, self.style)

    def undo(self, document: Document) -> None:
        if self.previous_text is None:
            raise ValueError("Nothing to undo for this operation")
        block = _find_block(document, self.block_id)
        block.set_plain_text(self.previous_text, self.previous_style)


class DeleteTextOperation(EditOperation):
    def __init__(self, block_id: str, start_offset: int, end_offset: int, style: Style):
        self.block_id = block_id
        self.start_offset = start_offset
        self.end_offset = end_offset
        self.style = style
        self.previous_text: Optional[str] = None
        self.previous_style: Optional[Style] = None

    def apply(self, document: Document) -> None:
        block = _find_block(document, self.block_id)
        full_text = block.get_plain_text()

        if self.start_offset < 0 or self.end_offset > len(full_text) or self.start_offset > self.end_offset:
            raise ValueError("Invalid delete range")

        self.previous_text = full_text
        self.previous_style = block.runs[0].style if block.runs else Style()

        new_text = full_text[:self.start_offset] + full_text[self.end_offset:]
        block.set_plain_text(new_text, self.style)

    def undo(self, document: Document) -> None:
        if self.previous_text is None:
            raise ValueError("Nothing to undo for this operation")
        block = _find_block(document, self.block_id)
        block.set_plain_text(self.previous_text, self.previous_style)


class ApplyStyleOperation(EditOperation):
    def __init__(self, block_id: str, style: Style):
        self.block_id = block_id
        self.style = style
        self.previous_text: Optional[str] = None
        self.previous_style: Optional[Style] = None

    def apply(self, document: Document) -> None:
        block = _find_block(document, self.block_id)
        self.previous_text = block.get_plain_text()
        self.previous_style = block.runs[0].style if block.runs else Style()
        block.set_plain_text(self.previous_text, self.style)

    def undo(self, document: Document) -> None:
        if self.previous_text is None:
            raise ValueError("Nothing to undo for this operation")
        block = _find_block(document, self.block_id)
        block.set_plain_text(self.previous_text, self.previous_style)


def _find_block(document: Document, block_id: str) -> Block:
    block = next((b for b in document.blocks if b.block_id == block_id), None)
    if block is None:
        raise ValueError("Block not found")
    return block


# =========================
# Document Service
# =========================

class DocumentService:
    def __init__(self, collaboration_engine: Optional[CollaborationEngine] = None):
        self.documents: Dict[str, Document] = {}
        self.collaboration_engine = collaboration_engine or SimpleCollaborationEngine()
        self.undo_stack: Dict[str, List[EditOperation]] = {}
        self.redo_stack: Dict[str, List[EditOperation]] = {}

    def create_document(self, owner: User, title: str) -> Document:
        doc = Document(
            doc_id=str(uuid.uuid4()),
            title=title,
            owner=owner,
        )
        self.documents[doc.doc_id] = doc
        self.undo_stack[doc.doc_id] = []
        self.redo_stack[doc.doc_id] = []
        return doc

    def get_document(self, doc_id: str) -> Document:
        if doc_id not in self.documents:
            raise ValueError("Document not found")
        return self.documents[doc_id]

    def add_block(
        self,
        doc_id: str,
        user: User,
        block_type: BlockType,
        text: str,
        style: Style,
    ) -> Block:
        doc = self.get_document(doc_id)
        if not doc.can_edit(user):
            raise PermissionError("User cannot edit")

        block = Block(
            block_id=str(uuid.uuid4()),
            block_type=block_type,
            runs=[TextRun(text, style)],
        )
        doc.blocks.append(block)
        return block

    def apply_operation(
        self,
        doc_id: str,
        user: User,
        operation: EditOperation,
        mode: EditMode = EditMode.EDIT,
    ) -> Optional[Suggestion]:
        doc = self.get_document(doc_id)
        if not doc.can_edit(user):
            raise PermissionError("User cannot edit")

        if mode == EditMode.SUGGEST:
            suggestion = Suggestion(
                suggestion_id=str(uuid.uuid4()),
                suggested_by=user,
                operation=operation,
            )
            doc.suggestions.append(suggestion)
            return suggestion

        self.collaboration_engine.transform_and_apply(doc, operation)
        self.undo_stack[doc_id].append(operation)
        self.redo_stack[doc_id].clear()
        return None

    def undo(self, doc_id: str, user: User) -> None:
        doc = self.get_document(doc_id)
        if not doc.can_edit(user):
            raise PermissionError("User cannot edit")

        if not self.undo_stack[doc_id]:
            raise ValueError("Nothing to undo")

        operation = self.undo_stack[doc_id].pop()
        operation.undo(doc)
        self.redo_stack[doc_id].append(operation)

    def redo(self, doc_id: str, user: User) -> None:
        doc = self.get_document(doc_id)
        if not doc.can_edit(user):
            raise PermissionError("User cannot edit")

        if not self.redo_stack[doc_id]:
            raise ValueError("Nothing to redo")

        operation = self.redo_stack[doc_id].pop()
        self.collaboration_engine.transform_and_apply(doc, operation)
        self.undo_stack[doc_id].append(operation)

    def accept_suggestion(self, doc_id: str, user: User, suggestion_id: str) -> None:
        doc = self.get_document(doc_id)
        if not doc.can_edit(user):
            raise PermissionError("User cannot accept suggestion")

        suggestion = next((s for s in doc.suggestions if s.suggestion_id == suggestion_id), None)
        if suggestion is None:
            raise ValueError("Suggestion not found")
        if suggestion.status != SuggestionStatus.PENDING:
            raise ValueError("Suggestion is not pending")

        self.collaboration_engine.transform_and_apply(doc, suggestion.operation)
        self.undo_stack[doc_id].append(suggestion.operation)
        self.redo_stack[doc_id].clear()
        suggestion.status = SuggestionStatus.ACCEPTED

    def reject_suggestion(self, doc_id: str, user: User, suggestion_id: str) -> None:
        doc = self.get_document(doc_id)
        if not doc.can_edit(user):
            raise PermissionError("User cannot reject suggestion")

        suggestion = next((s for s in doc.suggestions if s.suggestion_id == suggestion_id), None)
        if suggestion is None:
            raise ValueError("Suggestion not found")
        if suggestion.status != SuggestionStatus.PENDING:
            raise ValueError("Suggestion is not pending")

        suggestion.status = SuggestionStatus.REJECTED

    def add_comment(
        self,
        doc_id: str,
        user: User,
        block_id: str,
        start_offset: int,
        end_offset: int,
        text: str,
    ) -> Comment:
        doc = self.get_document(doc_id)
        if not doc.can_comment(user):
            raise PermissionError("User cannot comment")

        block = _find_block(doc, block_id)
        full_text = block.get_plain_text()
        if start_offset < 0 or end_offset > len(full_text) or start_offset > end_offset:
            raise ValueError("Invalid comment range")

        comment = Comment(
            comment_id=str(uuid.uuid4()),
            author=user,
            block_id=block_id,
            start_offset=start_offset,
            end_offset=end_offset,
            text=text,
        )
        doc.comments.append(comment)
        return comment

    def add_comment_reply(
        self,
        doc_id: str,
        user: User,
        comment_id: str,
        text: str,
    ) -> CommentReply:
        doc = self.get_document(doc_id)
        if not doc.can_comment(user):
            raise PermissionError("User cannot reply to comments")

        comment = next((c for c in doc.comments if c.comment_id == comment_id), None)
        if comment is None:
            raise ValueError("Comment not found")

        return comment.add_reply(user, text)

    def resolve_comment(self, doc_id: str, user: User, comment_id: str) -> None:
        doc = self.get_document(doc_id)
        if not doc.can_comment(user):
            raise PermissionError("User cannot resolve comments")

        comment = next((c for c in doc.comments if c.comment_id == comment_id), None)
        if comment is None:
            raise ValueError("Comment not found")

        comment.resolve()

    def share_document(
        self,
        doc_id: str,
        owner: User,
        target_user: User,
        role: PermissionRole,
    ) -> None:
        doc = self.get_document(doc_id)
        if doc.owner.user_id != owner.user_id:
            raise PermissionError("Only owner can share")

        doc.add_permission(Permission(target_user, role))

    def save_version(self, doc_id: str, user: User) -> Version:
        doc = self.get_document(doc_id)
        if not doc.can_edit(user):
            raise PermissionError("User cannot save version")

        snapshot = copy.deepcopy(doc.blocks)
        version = Version(
            version_id=str(uuid.uuid4()),
            created_by=user,
            snapshot_blocks=snapshot,
        )
        doc.versions.append(version)
        return version

    def get_versions(self, doc_id: str, user: User) -> List[Version]:
        doc = self.get_document(doc_id)
        if not doc.can_view(user):
            raise PermissionError("User cannot view versions")
        return list(doc.versions)

    def restore_version(self, doc_id: str, user: User, version_id: str) -> None:
        doc = self.get_document(doc_id)
        if not doc.can_edit(user):
            raise PermissionError("User cannot restore version")

        version = next((v for v in doc.versions if v.version_id == version_id), None)
        if version is None:
            raise ValueError("Version not found")

        doc.blocks = copy.deepcopy(version.snapshot_blocks)

    def join_session(
        self,
        doc_id: str,
        user: User,
        selection_range: SelectionRange,
    ) -> CollaborationSession:
        doc = self.get_document(doc_id)
        if not doc.can_view(user):
            raise PermissionError("User cannot access document")

        session = CollaborationSession(
            session_id=str(uuid.uuid4()),
            user=user,
            selection_range=selection_range,
        )
        doc.active_sessions[user.user_id] = session
        return session

    def update_selection(
        self,
        doc_id: str,
        user: User,
        selection_range: SelectionRange,
    ) -> None:
        doc = self.get_document(doc_id)
        session = doc.active_sessions.get(user.user_id)
        if session is None:
            raise ValueError("Session not found")

        session.selection_range = selection_range

    def leave_session(self, doc_id: str, user: User) -> None:
        doc = self.get_document(doc_id)
        session = doc.active_sessions.get(user.user_id)
        if session:
            session.leave()
            del doc.active_sessions[user.user_id]

    def get_active_sessions(self, doc_id: str, user: User) -> List[CollaborationSession]:
        doc = self.get_document(doc_id)
        if not doc.can_view(user):
            raise PermissionError("User cannot view active sessions")
        return list(doc.active_sessions.values())


# =========================
# Demo
# =========================

if __name__ == "__main__":
    service = DocumentService()

    owner = User("u1", "Alice", "alice@example.com")
    editor = User("u2", "Bob", "bob@example.com")
    commenter = User("u3", "Charlie", "charlie@example.com")
    viewer = User("u4", "David", "david@example.com")

    doc = service.create_document(owner, "Google Docs Advanced LLD")

    service.share_document(doc.doc_id, owner, editor, PermissionRole.EDITOR)
    service.share_document(doc.doc_id, owner, commenter, PermissionRole.COMMENTER)
    service.share_document(doc.doc_id, owner, viewer, PermissionRole.VIEWER)

    normal_style = Style()
    heading_style = Style(bold=True, font_size=18)
    emphasis_style = Style(italic=True, color="#1a73e8")

    heading = service.add_block(
        doc.doc_id,
        owner,
        BlockType.HEADING,
        "System Design Notes",
        heading_style,
    )

    para = service.add_block(
        doc.doc_id,
        owner,
        BlockType.PARAGRAPH,
        "Initial content here.",
        normal_style,
    )

    # normal edit
    service.apply_operation(
        doc.doc_id,
        editor,
        InsertTextOperation(
            para.block_id,
            len("Initial content here."),
            " More text added by editor.",
            normal_style,
        ),
        mode=EditMode.EDIT,
    )

    print("After edit:", para.get_plain_text())

    # undo / redo
    service.undo(doc.doc_id, editor)
    print("After undo:", para.get_plain_text())

    service.redo(doc.doc_id, editor)
    print("After redo:", para.get_plain_text())

    # suggestion mode
    suggestion = service.apply_operation(
        doc.doc_id,
        editor,
        InsertTextOperation(
            para.block_id,
            0,
            "Suggested: ",
            normal_style,
        ),
        mode=EditMode.SUGGEST,
    )
    print("Suggestion created:", suggestion.suggestion_id, suggestion.status.value)

    service.accept_suggestion(doc.doc_id, owner, suggestion.suggestion_id)
    print("After accepting suggestion:", para.get_plain_text())

    # comments and replies
    comment = service.add_comment(
        doc.doc_id,
        commenter,
        para.block_id,
        0,
        9,
        "Can we improve this intro?",
    )
    reply = service.add_comment_reply(
        doc.doc_id,
        owner,
        comment.comment_id,
        "Yes, I will refine it.",
    )

    print("Comment text:", comment.text)
    print("Replies:", len(comment.replies))
    print("First reply:", reply.text)

    # version save
    version = service.save_version(doc.doc_id, owner)
    print("Saved version:", version.version_id)

    # collaboration sessions with selection range
    session_owner = service.join_session(
        doc.doc_id,
        owner,
        SelectionRange(block_id=heading.block_id, start_offset=5, end_offset=5),
    )

    session_editor = service.join_session(
        doc.doc_id,
        editor,
        SelectionRange(block_id=para.block_id, start_offset=0, end_offset=12),
    )

    service.update_selection(
        doc.doc_id,
        editor,
        SelectionRange(block_id=para.block_id, start_offset=10, end_offset=20),
    )

    print("Document title:", doc.title)
    print("Blocks:", len(doc.blocks))
    print("Comments:", len(doc.comments))
    print("Suggestions:", len(doc.suggestions))
    print("Versions:", len(doc.versions))
    print("Active sessions:", len(doc.active_sessions))
    print("Viewer can edit:", service.get_document(doc.doc_id).can_edit(viewer))

    print("\nActive selections:")
    for s in service.get_active_sessions(doc.doc_id, owner):
        sel = s.selection_range
        print(s.user.name, sel.block_id, sel.start_offset, sel.end_offset)
```

## Design Principles, SOLID Principles, OOP Concepts, and Design Patterns Used in this Google Docs LLD

# 1. Design Principles Used

## 1.1 Separation of Concerns

Different responsibilities are split across different classes.

Examples:
- `Document` stores document state
- `EditOperation` handles edit behavior
- `Comment` handles comment data
- `Version` handles version snapshots
- `CollaborationSession` handles active user presence
- `DocumentService` coordinates workflows
- `CollaborationEngine` is reserved for concurrency logic like OT or CRDT

This keeps the system easier to understand, maintain, and extend.

---

## 1.2 Composition Over Inheritance

The design prefers building objects from smaller pieces instead of deep inheritance trees.

Examples:
- `Document` contains `Block`
- `Block` contains `TextRun`
- `Comment` contains `CommentReply`
- `Document` contains `Suggestion`, `Version`, and `CollaborationSession`

This keeps the model flexible and modular.

---

## 1.3 Command Oriented Design

Editing is modeled as operations:

- `InsertTextOperation`
- `DeleteTextOperation`
- `ApplyStyleOperation`

This is powerful because operations become first class objects and can support:
- apply
- undo
- redo
- suggestion mode
- future collaboration transforms

---

## 1.4 Encapsulation of Business Rules

Permission checks are encapsulated inside `Document` methods like:

- `can_view`
- `can_comment`
- `can_edit`

That is better than scattering permission logic across multiple classes.

---

## 1.5 Extensibility

The design is built so new features can be added without breaking the core model.

Examples:
- new block types
- new edit operations
- new collaboration engines
- richer formatting models
- new permission roles

---

## 1.6 Layered Coordination

`DocumentService` acts as a coordinator layer between users and document entities.

That keeps entity classes from becoming too workflow heavy and centralizes document related use cases.

---

# 2. SOLID Principles Used

## 2.1 Single Responsibility Principle (SRP)

Each class mostly has one reason to change.

Examples:
- `Style` only models formatting
- `TextRun` only models text plus style
- `Comment` only models comments
- `Suggestion` only models suggestion state
- `EditOperation` subclasses only perform document edits
- `DocumentService` coordinates use cases

This makes the system easier to test, reason about, and maintain.

---

## 2.2 Open Closed Principle (OCP)

The design is open for extension without modifying stable code.

Examples:
- add a new edit operation by creating another `EditOperation` subclass
- add a new collaboration engine by implementing `CollaborationEngine`
- add a new block type in `BlockType`
- add new permission handling rules with minimal service changes

This makes the design future friendly.

---

## 2.3 Liskov Substitution Principle (LSP)

All subclasses of `EditOperation` can be used wherever an `EditOperation` is expected.

Examples:
- `InsertTextOperation`
- `DeleteTextOperation`
- `ApplyStyleOperation`

Also, any future `CollaborationEngine` implementation can replace `SimpleCollaborationEngine`.

This ensures substitutability without breaking client code.

---

## 2.4 Interface Segregation Principle (ISP)

Interfaces are small and focused.

Examples:
- `EditOperation` only has `apply` and `undo`
- `CollaborationEngine` only exposes `transform_and_apply`

Clients are not forced to depend on methods they do not use.

---

## 2.5 Dependency Inversion Principle (DIP)

This is partially followed.

Good part:
- `DocumentService` depends on the abstraction `CollaborationEngine`, not directly on one concrete engine behavior

Could improve further:
- repositories or storage abstractions are not yet extracted
- persistence and session broadcasting are still in memory and implicit

Still, the collaboration layer already shows DIP thinking.

---

# 3. OOP Concepts Used

## 3.1 Encapsulation

Classes bundle data and behavior together.

Examples:
- `Block` knows how to get plain text
- `Comment` knows how to resolve itself
- `CollaborationSession` knows how to leave
- `Document` knows how to evaluate user permissions

This protects internal logic and keeps the model clean.

---

## 3.2 Abstraction

Abstract base classes hide implementation details.

Examples:
- `EditOperation`
- `CollaborationEngine`

The caller only knows what an operation should do, not how each implementation works internally.

---

## 3.3 Inheritance

Used in a focused and controlled way.

Examples:
- concrete edit operations inherit from `EditOperation`
- `SimpleCollaborationEngine` implements `CollaborationEngine`

This supports reuse of shared contracts.

---

## 3.4 Polymorphism

The same method call behaves differently based on the concrete object.

Example:
- `operation.apply(document)` works for insert, delete, and style operations

That is runtime polymorphism and makes the editor extensible.

---

## 3.5 Composition

The system is heavily composition based.

Examples:
- `Document` has blocks, comments, versions, sessions, suggestions
- `Block` has runs
- `Comment` has replies

This is one of the strongest OOP qualities of the design.

---

# 4. Design Patterns Used

## 4.1 Command Pattern

**Where used**
- `EditOperation`
- `InsertTextOperation`
- `DeleteTextOperation`
- `ApplyStyleOperation`

**Why it fits**
Each edit is wrapped as an object.

That allows:
- executing edits
- undoing edits
- redoing edits
- storing edits as suggestions
- future transformation by OT or CRDT engines

This is a strong fit for Command pattern.

---

## 4.2 Facade Pattern

**Where used**
- `DocumentService`

**Why it fits**
It provides one entry point for:
- create document
- edit
- comment
- share
- version
- sessions
- suggestions
- undo redo

Clients do not need to directly coordinate all lower level objects.

This simplifies system usage.

---

## 4.3 Strategy Pattern

**Where used**
- `CollaborationEngine`
- `SimpleCollaborationEngine`

**Why it fits**
The collaboration behavior can be swapped later with:
- `SimpleCollaborationEngine`
- `OTCollaborationEngine`
- `CRDTCollaborationEngine`

without changing higher level service workflow.

This makes collaboration logic pluggable.

---

## 4.4 Composite Style Hierarchical Modeling

**Where used**
- `Document`
- `Block`
- `TextRun`

**Why it fits**
The document structure is hierarchical:

- document
- block
- text runs

This is not a strict textbook Composite implementation, but it follows composite style modeling where complex structures are built from nested components.

---

# 5. Best Interview Summary

You can say:

> This Google Docs LLD uses strong separation of concerns and composition based modeling. The main design patterns are Command for edit operations, Facade through `DocumentService`, and Strategy through the collaboration engine abstraction. It follows SOLID principles well, especially SRP, OCP, and LSP. In OOP terms, it uses encapsulation, abstraction, inheritance, polymorphism, and composition to keep the editor modular and extensible.