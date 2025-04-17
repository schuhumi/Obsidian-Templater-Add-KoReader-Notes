<%*
/* CONFIG */
const LOG_PREFIX = "KoReader Note converter:" // for console.log
function note2sidecar(note) {
	// Infer the KoReader sidecar folder path from the Obsidian
	// note where the user wants to insert the KoReader notes into.
	// Example:
	// Let's assume the user has a folder Uni/Literatur/PDFs where the
	// literature pdfs and also the KoReader sidecar files are
	// (KoReader's default is to store them beside the books). The
	// corresponding notes are in the folder Uni/Literatur/Notizen.
	// Also assume the following naming convention:
	// - the pdf is named surname2021.pdf for example (typical citation key style)
	// - KoReader's sidecar folder is surname2021.sdr
	// - the corresponding obsidian note is @surname2021.md
	
	if ((note.parent.path != "Uni/Literatur/Notizen") || !note.basename.startsWith("@") ) {
		console.log(LOG_PREFIX, note.path, "is not a literature note")
		return null
	}
	// We do not need to check if this exists, that happens after calling this function
	return "Uni/Literatur/PDFs/" + note.basename.substring(1) + ".sdr"
}
/* END CONFIG */


function load_koreader_sdr_lua(lua) {
	if (!lua)
		return null
	// A hacky way of interpreting KoReaders metadata (lua) as json
	const json = lua
		.replace(/--\[\[(.|\n)*\]\]/g, '') // remove multiline comments
		.replace(/--.*/g,'')  // remove single line comments
		.replace('return {\n', '{\n')  // remove return statement
		.replace(/\\\n/g, '\\n')  // write multiline strings using \n
		.replace(/\[|\]/g, '') // remove the brackets
		.replace(/(?<!["'])\b\d+(?:\.\d+)?(?=\s*=\s*)/g, '"$&"')  // convert int keys to strings
		.replace(/=/g, ':') // replace "=" with ":"
		.replace(/(\,)(?=\s*})/g, '') // remove trailing commas
	return JSON.parse(json)
}

function format_koreader_json_highlights(annotations) {
	let output = ""
	for (key in annotations) {
		entry = annotations[key]
		if (entry.text) {
			output += `> [!quote] <span style="color: ${
					entry.color
				}">Page ${
					entry.page
				}, on ${
					entry.datetime_updated || entry.datetime
				}</span>\n> ${
					entry.text.split("\n").join("\n> ")
				}\n\n`;
			if (entry.note) {
				output += `${entry.note}\n\n\n`;
			}
		}
	}
	return output;
}

async function get_koreader_metadata_from_sdr_folder_path(path) {
	const sdr_folder = tp.app.vault.getFolderByPath(path)
	if (!sdr_folder) {
		console.log(LOG_PREFIX, "KoReader sidecar folder not found for path:", path)
		return null
	}
	const metadata_file = sdr_folder.children.filter(i => i.name == "metadata.pdf.lua")[0]
	if (!metadata_file) {
		console.log(LOG_PREFIX, "KoReader sidecar file metadata.pdf.lua not found in:", path)
		return null
	}
	return tp.app.vault.readRaw(metadata_file.path)
}

const note = tp.app.workspace.getActiveFile()
console.log(LOG_PREFIX, "Called from:", note.path)
sdr_folder_path = note2sidecar(note)
console.log(LOG_PREFIX, "Corresponding sidecar location:", sdr_folder_path)
if (sdr_folder_path) {
	const content_lua = await get_koreader_metadata_from_sdr_folder_path(sdr_folder_path)
	const content = load_koreader_sdr_lua(content_lua)
	if (content.annotations) {
		return format_koreader_json_highlights(content.annotations)
	} else {
		console.log(LOG_PREFIX, "Couldn't retrieve .annotations from:", content)
	}
} else {
	console.log(LOG_PREFIX, "This is not a literature note")
}
%>
