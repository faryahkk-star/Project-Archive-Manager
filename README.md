import csv
import shutil
from pathlib import Path
from datetime import datetime, timedelta


class ProjectArchiveManager:
    def __init__(self, workspace):
        self.workspace = Path(workspace)
        self.archive = self.workspace / "archive"
        self.archive.mkdir(exist_ok=True)

    def last_modified(self, folder):
        latest = datetime.fromtimestamp(0)

        for item in folder.rglob("*"):
            try:
                modified = datetime.fromtimestamp(item.stat().st_mtime)
                if modified > latest:
                    latest = modified
            except OSError:
                continue

        return latest

    def archive_old_projects(self, days=90):
        threshold = datetime.now() - timedelta(days=days)
        archived = []

        for project in self.workspace.iterdir():

            if not project.is_dir():
                continue

            if project.name == "archive":
                continue

            last_change = self.last_modified(project)

            if last_change < threshold:
                destination = self.archive / project.name

                if destination.exists():
                    continue

                shutil.move(str(project), str(destination))

                archived.append({
                    "project": project.name,
                    "last_modified": last_change.strftime("%Y-%m-%d"),
                    "archived_at": datetime.now().strftime("%Y-%m-%d")
                })

        return archived

    def save_report(self, archived):
        report = self.workspace / "archive_report.csv"

        with open(report, "w", newline="", encoding="utf-8") as file:
            writer = csv.DictWriter(
                file,
                fieldnames=["project", "last_modified", "archived_at"]
            )

            writer.writeheader()

            for row in archived:
                writer.writerow(row)

        return report


def main():
    workspace = input("Workspace path: ").strip()

    manager = ProjectArchiveManager(workspace)

    days = input("Archive projects inactive for how many days? (default=90): ").strip()

    days = int(days) if days else 90

    archived = manager.archive_old_projects(days)

    if not archived:
        print("\nNo old projects found.")
        return

    report = manager.save_report(archived)

    print(f"\nArchived {len(archived)} project(s).")
    print(f"Report saved to: {report}")


if __name__ == "__main__":
    main()
