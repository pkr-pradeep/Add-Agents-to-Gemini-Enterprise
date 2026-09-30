# Add-Agents-to-Gemini-Enterprise
# 1. Download the history from GitHub without merging
git fetch origin

# 2. Point your local Git history to match GitHub's history 
# (Your actual files in the folder will not be deleted or modified)
git reset origin/main

# 3. Stage, commit, and push your changes cleanly
git add .
git commit -m "Update files with new changes"
git push -u origin main
