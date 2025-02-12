<%*
// Prompt the user for a citation key, showing suggestions from the "Reference Notes" folder.
const referenceNotesFolder = "Reference Notes";
const referenceNotes = await app.vault.getFiles()
  .filter(f => f.path.startsWith(referenceNotesFolder) && f.extension === "md");

// Map reference notes to include aliases in the suggestions.
const suggestions = referenceNotes.map(note => {
  const frontmatter = app.metadataCache.getFileCache(note)?.frontmatter || {};
  const aliases = frontmatter.aliases || [];
  return {
    name: note.basename,
    display: `${note.basename}${aliases.length ? ` (aliases: ${aliases.join(", ")})` : ""}`,
    aliases: aliases
  };
});

// Use tp.system.suggester to select a citation key.
const selected = await tp.system.suggester(
  suggestions.map(s => s.display),
  suggestions
);
const citationKey = selected?.name;

if (!citationKey) {
  throw new Error("No citation key provided. Aborting template creation.");
}

// Prompt the user for a folder to save the literature note, with a default suggestion.
const defaultFolder = "03 Literature Notes";
const folderChoices = [defaultFolder, ...new Set(app.vault.getAllLoadedFiles().map(f => f.parent?.path).filter(Boolean))];
const selectedFolder = await tp.system.suggester(folderChoices, folderChoices) || defaultFolder;

// Construct the path for the new literature note with @.
const filePath = `${selectedFolder}/@${citationKey}.md`;

// Locate the reference note corresponding to the selected citation key.
const referenceNote = referenceNotes.find(note => note.basename === citationKey);
if (!referenceNote) {
  throw new Error(`Reference note for citation key '${citationKey}' not found.`);
}

// Read the frontmatter of the reference note to extract the "aliases" property.
const referenceFrontmatter = await app.metadataCache.getFileCache(referenceNote)?.frontmatter || {};
const aliases = referenceFrontmatter.aliases?.map((alias) => `${alias} (Literature Note)`) || [];
const tags = ["#literature_note"];
const referenceNoteLink = `[[${referenceNote.basename}]]`;

// Check if the file already exists.
const existingFile = app.vault.getAbstractFileByPath(filePath);
if (existingFile) {
  // If the file exists, update only the YAML frontmatter fields that already exist.
  const existingContent = await app.vault.read(existingFile);
  const existingFrontmatterMatch = existingContent.match(/^---[\s\S]*?---/);
  let existingFrontmatter = {};
  if (existingFrontmatterMatch) {
    existingFrontmatter = app.metadataCache.getFileCache(existingFile)?.frontmatter || {};
  }

  // Merge frontmatter fields, preserving non-overlapping fields.
  let updatedTags = tags;
  if (existingFrontmatter.tags) {
    updatedTags = Array.isArray(existingFrontmatter.tags) ? existingFrontmatter.tags : [existingFrontmatter.tags];
    if (!updatedTags.includes(tags[0])) {
      updatedTags.push(tags[0]);
    }
  }

  const updatedFrontmatter = {
    ...existingFrontmatter,
    "reference-note": `"${referenceNoteLink}"`,
    aliases: aliases.length ? aliases : existingFrontmatter.aliases,
    tags: updatedTags
  };

  const frontmatterString = `---\n${Object.entries(updatedFrontmatter)
    .map(([key, value]) => `${key}: ${Array.isArray(value) ? JSON.stringify(value) : value}`)
    .join("\n")}\n---`;

  const updatedContent = existingFrontmatterMatch
    ? existingContent.replace(/^---[\s\S]*?---/, frontmatterString)
    : `${frontmatterString}\n${existingContent}`;

  await app.vault.modify(existingFile, updatedContent);

  // Switch to the existing tab if the file is already open.
  const existingTab = app.workspace.getLeavesOfType("markdown").find(leaf => leaf.getDisplayText() === `@${citationKey}`);
  if (existingTab) {
    app.workspace.revealLeaf(existingTab);
  } else {
    await app.workspace.openLinkText(filePath, "", true);
  }
} else {
  // If the file does not exist, create it with the full template.
  const newContent = `---\naliases: ${JSON.stringify(aliases)}\nreference-note: "${referenceNoteLink}"\ntags: ${JSON.stringify(tags)}\n---\n## Summary\n## Key Ideas\n## Connections\n## Reflections\n`;
  await app.vault.create(filePath, newContent);
  await app.workspace.openLinkText(filePath, "", true);
}
return null;
%>
